@def title = "Multimodal posteriors Part 2"
@def published = "31 December 2020"
@def tags = ["stats", "deep learning"]
@def hasmath = true

# Multimodality Part 2: Deep Learning

- Are neural networks misspecified? Wilson et al. argue yes but I am not so sure. 
    If you have millions of parameters then your model space is either extremely 
    large or there are some redundancies in your parameters. Both could be true.
- Yuling Yao: Neural networks have such a terrible posterior geometry that it is 
    incredibly difficult to get an accurate representation of the posterior. 
    Therefore any claim that the Bayesian posterior in neural networks is suboptimal 
    for prediction must be carefully evaluated because it might just be a failure 
    of the inference algorithm.

Main argument:
The main distinguishing feature of being Bayesian is marginalisation i.e. 
considering multiple possible parameter settings in the predictive distribution.
Taking this view Bayesianism is extremely helpful for deeply learning because 
these models are inherently misspecified and there exist very different sets of 
parameters which provide similar predictive performance. Ergo people should 
average over different parameters when making predictions. Simply using a point 
estimate is not enough.

The following challenges are identified:
- Finding a diverse set of parameters which give rise to functionally different 
    predictions but with similar predictive performance
    - This has connections to deep ensembles and bayesian model averaging
- We still need to give sensible priors for our architectures to make sure that 
    our posterior isn't diffuse this means we need the right *inductive biases*.
    - Empirical results suggest the **NN architecture** is a good place to work on 
        building better priors (i.e. build in invariance); the priors on the weights 
        does not matter that much in comparison

## Open questions

- Neural network architectures are usually discovered and built on a trial and 
    error basis. Could we develop a principled workflow akin to Betancourt or 
    Gelman et al which brings more rigour into neural network design?

- How can we do prior predictive checks for NNs? NN predictions from the prior 
    will usually be nonsensical anyway.
    - Maybe we can reuse posteriors from off-the-shelf architectures?
    - How could we properly transfer priors between different architectures?

- As a long-term goal we want to use NN predictions as part of a bigger Bayesian 
    model. For example, we might have some tabular data about the economic 
    performance of a country and at the same time also a NN model that counts 
    the number of cars parked in front of supermarkets from satellite images.
    Now we want to use the quantity of estimated cars and we want to account for 
    uncertainty. Ideally, we only specify the "forward" model once in some Julia, 
    Python code and the separately we can specify how to fit the individual 
    components. This would heavily rely on the programmable inference ideas 
    from Gen