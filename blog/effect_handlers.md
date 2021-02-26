@def title = "Building a PPL: Effect Handlers"
@def hasmath = true
@def hascode = true
@def published = "25 December 2020"
@def tags = ["julia", "PPL"]

TODO: Add RSS title.

# Building a PPL: Effect Handlers

Goals with this blog series:

1. Show that effect handlers are a powerful abstraction for implementing PPl
2. Show that Julia can easily implement high-level DSLs
3. Show that Julia is very modular

One blog post for each point. Especially points 2 and 3 make Julia extremely
suitable for scientific computing.

NOTE: Do not focus on performance in this implementation.

Do not assume any Julia knowledge.

## Target language

Describe the language we are going to implement. Use
[Colin Caroll's tour of PPL APIs](https://colcarroll.github.io/ppl-api/)
model as a showcase.

```julia
# TODO: Change sample interface.
@jyro function model(xs)
    D = size(xs)[2]
    w ~ MvNormal(zeros(D), I)
    y ~ MvNormal(xs * w, 0.1*I)
end

condition!(model, Dict(:y => y_obs))
samples, stats = hmc(model, 0.05, 10, 1000, (X,));
```

## What is a PPL?

Use similar introduction as in Gen and Pyro to motivate PPL. Want to track random
choices in a generative function.

Code example: Same forward function as in previous section but just using rand calls.

```julia
function model(xs)
    D = size(xs)[2]
    w = rand(MvNormal(zeros(D), I))
    y = rand(MvNormal(xs * w, 0.1*I))
    return y
end
```

Things we might want to do:

- Evaluate log density function.
- Track results of random choices inside `model`
- Conditioned on an observed `y` we want to infer `w`.

## What are effect handlers?

Refer to Pyro effect handlers tutorial and the PPL effect handler paper.

Refer to more functional CSP style implementation resources (i.e. WebPPL and
Mike Innes Effect Handlers).

Explain different concepts that can be implemented with effect handlers.

### Effect handlers in Julia

Explain messaging implementation.

Code example: First generative functions with all the handler code.

```julia
abstract type AbstractHandler end

function enter!(trace::Dict, h::AbstractHandler)
    return
end

function exit!(trace::Dict, h::AbstractHandler)
    return
end

function process_message!(trace::Dict, h::AbstractHandler, msg::Dict)
    return
end

function postprocess_message!(trace::Dict, h::AbstractHandler, msg::Dict)
    return
end
```

Can build effectful sample function:

```julia
function sample!(
    trace::Dict,
    handlers_stack::Array{AbstractHandler,1},
    name::Any,
    dist::Distributions.Distribution
)
    if length(handlers_stack) == 0
        return rand(dist)
    end

    # NOTE: In this simplified case we could replace :args with :dist because
    #       our "effect" is always sampling from a distribution.
    initial_msg = Dict(
        :fn => rand,
        :args => (dist, ),
        :name => name,
        :value => nothing,
        :is_observed => false,
        :done => false, # Use if we still want other handlers to be active.
        :stop => false, # Use if we want handlers below on the stack not to be active.
        :continuation => nothing
    )
    msg = apply_stack!(trace, handlers_stack, initial_msg)
    return msg[:value]
end
```

Apply effect handlers as follows:

```julia
function apply_stack!(
    trace::Dict,
    handlers_stack::Array{AbstractHandler,1},
    msg::Dict
)
    # NOTE: This function assumes handlers_stack is not empty.
    handler_counter = 0
    # Loop through handlers from top of the stack to the bottom.
    for handler in handlers_stack[end:-1:1]
        handler_counter += 1
        process_message!(trace, handler, msg)
        if msg[:stop]
            break
        end
    end

    if !(msg[:value] != nothing || msg[:done])
        msg[:value] = msg[:fn](msg[:args]...)
    end

    # Loop through handlers from bottom of the stack to the top.
    # If we exited the first loop early then we will start looping from the
    # handler which caused the loop to exit.
    for handler in handlers_stack[end-handler_counter+1:end]
        postprocess_message!(trace, handler, msg)
    end

    if msg[:continuation] != nothing
        msg[:continuation](trace, msg)
    end

    return msg
end
```

Trace effect handler:

```julia
struct TraceHandler <: AbstractHandler end

function enter!(trace::Dict, h::TraceHandler)
    trace[:msgs] = Dict()
end

function postprocess_message!(trace::Dict, h::TraceHandler, msg::Dict)
    trace[:msgs][msg[:name]] = copy(msg)
end
```

LogJoint handler:

```julia
struct LogJointHandler <: AbstractHandler end

function enter!(trace::Dict, h::LogJointHandler)
    trace[:logjoint] = 0.0
end

function postprocess_message!(trace::Dict, h::LogJointHandler, msg::Dict)
    dist = msg[:args][1]
    if msg[:value] != nothing
        trace[:logjoint] = trace[:logjoint] + logpdf(dist, msg[:value])
    else
        # TODO: Warning might be annoying when running the queue handler.
        @warn "Encountered site with value nothing." msg[:name]
    end
end
```

Condition Handler:

```julia
struct ConditionHandler <: AbstractHandler
    data::Dict
end

function process_message!(trace::Dict, h::ConditionHandler, msg::Dict)
    if haskey(h.data, msg[:name])
        msg[:value] = h.data[msg[:name]]
        msg[:stop] = true
        msg[:is_observed] = true
    end
end
```

## Final result

Code example: Full model with handler code.

### Extra section: Inference with discrete enumeration

Show how effect handlers can be used to implement sequential enumeration inference.

Effect handlers replay, queue and escape.

Hidden code blocks: Implementations of each handler.

Replay effect handler:

```julia
struct ReplayHandler <: AbstractHandler
    trace::Dict
end

function process_message!(trace::Dict, h::ReplayHandler, msg::Dict)
    if haskey(h.trace[:msgs], msg[:name])
        msg[:value] = h.trace[:msgs][msg[:name]][:value]
    end
end
```

Escape effect handler:

```julia
struct EscapeHandler <: AbstractHandler
    escape_fn::Function
end

struct EscapeException <: Exception
    trace::Dict
    name
end

function process_message!(trace::Dict, h::EscapeHandler, msg::Dict)
    if h.escape_fn(msg)
        msg[:done] = true
        cont(t, m) = throw(EscapeException(t, m[:name]))
        msg[:continuation] = cont
    end
end
```

Queue effect handler:

```julia
function queue(model::MinijyroModel, queue)
    max_tries = Int(1e6) # Make sure we do not have infinite loops.
    function _fn(handlers_stack, args...)
        # TODO: This cannot deal with the fact that model_fn might have been
        # been changed. Alternative is to pass model to the model_fn.
        m = MinijyroModel(handlers_stack, model.model_fn)
        for i in 1:max_tries
            top_trace = popfirst!(queue)
            try
                queue_m = handle(m, ReplayHandler(top_trace))
                # Escape when we see a new variable that is not in the trace.
                discr_esc(msg) = !haskey(top_trace[:msgs], msg[:name])
                queue_m = handle(queue_m, EscapeHandler(discr_esc))
                queue_m = handle(queue_m, TraceHandler())

                full_trace = queue_m(args...)
                # We only get to the return call if there has been no escape i.e.
                # if the top_trace has been a full trace.
                return full_trace
            catch e
                if isa(e, EscapeException)
                    for val in support(e.trace[:msgs][e.name][:args][1])
                        new_trace = deepcopy(e.trace)
                        new_trace[:msgs][e.name][:value] = val

                        # Make sure that we do not escape again when using this
                        # trace for replay.
                        new_trace[:msgs][e.name][:done] = false
                        new_trace[:msgs][e.name][:continuation] = nothing
                        push!(queue, new_trace)
                    end
                else
                    throw(e)
                end
            end
        end
    end

    return MinijyroModel(model.handlers_stack, _fn)
end
```

Explanation of inference algorithm. Point to WebPPL book for details.

Code example: Implementation of enumeration.

TODO: Should we have inference example in Part 3 or here? If we have it in Part 3
then we can use the "nice" syntax.
