@def title = "Building a PPL: Domain Specific Language"
@def published = "27 December 2020"
@def tags = ["julia", "PPL"]

# Building a PPL: Domain Specific Language

## Metaprogramming in Julia

Refer to [Julia docs](https://docs.julialang.org/en/v1/manual/metaprogramming/).

Allows to build high level interfaces.

## Minijyro Models

Motivation: Want to hide all the bookkeeping of trace and handler stack.

Want to keep track of handlers stack.

Definition of a model in our language:

```julia
struct MinijyroModel
    handlers_stack::Array{AbstractHandler,1}
    model_fn::Function
end

function (model::MinijyroModel)(args...)
    # TODO: Possibly give nicer error message when args are given in wrong type.
    return model.model_fn(model.handlers_stack, args...)
end
```

TODO: Full example of how a function gets translated.

## Using Macros to Build a DSL

Evaluate tilde calls to sample! statements:

```julia
function translate_tilde(expr)
    # NOTE: Code adapted from https://github.com/probcomp/Gen/blob/master/src/dsl/dsl.jl
    MacroTools.postwalk(expr) do e
        if MacroTools.@capture(e, {addr_} ~ rhs_)
            # NOTE: This does not sample
            :(Minijyro.sample!($TRACE_SYMBOL, $STACK_SYMBOL, $(addr), $rhs))
        elseif MacroTools.@capture(e, lhs_ ~ rhs_)
            name_symbol = QuoteNode(lhs)
            :($lhs = Minijyro.sample!($TRACE_SYMBOL, $STACK_SYMBOL, $name_symbol, $rhs))
        else
            e
        end
    end
end
```

Make sure we return a trace and keep track of any return statements that might exist.

```julia
function translate_return(expr, loop_var)
    MacroTools.postwalk(expr) do e
        if MacroTools.@capture(e, return r_)
            quote
                for $(loop_var) in $STACK_SYMBOL
                    exit!($TRACE_SYMBOL, $(loop_var))
                end
                $(TRACE_SYMBOL)[$(RETURN_KEY)] = $r
                return $TRACE_SYMBOL
            end
        else
            e
        end
    end
end
```

Putting everything together:

```julia
macro jyro(expr)
    fn_dict = MacroTools.splitdef(expr)
    pushfirst!(fn_dict[:args], STACK_SYMBOL)

    loop_var = gensym()
    body = translate_tilde(fn_dict[:body])
    body = translate_return(body, loop_var)

    fn_dict[:body] = quote
        $TRACE_SYMBOL = Dict{Any,Any}($(RETURN_KEY) => nothing)
        for $(loop_var) in $(STACK_SYMBOL)[end:-1:1]
            enter!($TRACE_SYMBOL, $(loop_var))
        end
        $(body.args...)
        # NOTE: If the last line of (body.args...) is already a return statement
        #       then the compiler is smart enough to remove this code.
        for $(loop_var) in $STACK_SYMBOL
            exit!($TRACE_SYMBOL, $(loop_var))
        end
        return $TRACE_SYMBOL
    end


    model_name = fn_dict[:name]
    fn_name = gensym(model_name)
    fn_dict[:name] = fn_name
    fn_dict[:rtype] = Dict{Any,Any} # Change return type of generated function to a Dict.
    fn_expr = esc(MacroTools.combinedef(fn_dict))

    return quote
        $fn_expr

        $(esc(model_name)) = MinijyroModel(AbstractHandler[], $(esc(fn_name)))
    end
end
```

TODO: Generated handler functions

@def title = "Building a PPL: Domain Specific Language"
@def hasmath = true
@def hascode = true

TODO: Add RSS title.