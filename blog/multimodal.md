@def title = "Multimodal posteriors"
@def published = "30 December 2020"
@def tags = ["julia", "stats"]
@def hasmath = true

# Multimodal posteriors

TODO: How to do references?

Summary of papers from Yao et al and Wilson et al.

Stacking paper:
- When do we have multimodal posteriors?
    - What type of multimodal posteriors do we have?
- Why not just run multiple chains and average over them?
    - That's essentialy what stacking does but it is important *how* you average
        over them
- What does stacking do?
    - Why is it not equivalent to Bayesian inference?
        - The stacking weights are computed by estimating the predictive performance
            on hold out data, instead of using the marginal likelihood or some 
            other Bayesian procedure.
    - Why does it give better predictive performance? Empirical 
        analysis or analytic proof?
        - The objective function is equivalent to minimising the predictive 
            error as the number of observations **and** the number of posterior 
            samples go to infinity
        - Analytic proof on a Cauchy toy example
            - Would be interesting to know whether these proofs generalise
            - Raises the general question: Can we approximate the true DG, with 
                a wrong model and non-mixing inferences?
                - Why is stacking able to do that?
                - Stacking is kind of like nonparametric inference because the 
                    number of clusters/modes is not specified beforehand
- What are the implications for neural nets?
    - It seems that in neural networks our models are underspecified and 
        (maybe?) wrong.
        - Using gazillion of parameters would imply that we can approximate almost
            any distribution so technically our model is not super wrong right?
    - Deep ensembles shouldn't average with equal weights
    - It might not make sense to use equal weighting of the different models


## When do we get multimodality?

We get multimodal posteriors either because

a. we do not have enough data
b. our model is misspecified.

In case a the multimodality will disappear with more data but b is an inherent 
problem.

Formally, if our model is correct then the Bernstein-von Mises theorem tells us 
that with enough data the posterior will concentrate around the true parameter 
value. However, what happens if our model is misspecified?

If the model is misspecified then the limiting predictive distribution is the 
distribution which minimises the KL between the true data generating process and 
our model

$$ 
\text{KL}(p_{true}(\cdot) || f(\cdot | \theta)) 
$$

Unfortunately, this reduces to a point estimate with more data. However, the 
key point about Bayesian inference is that we want to quantify the uncertainty 
in our inferences with probability so a point estimate defeats the purpose!

Instead we want a probability distribution on $\theta$ which minimises the 
predictive loss according to some divergence.

## Some examples

Potentially list the four examples using Turing models. For each of the 
four examples we have:

- A Turing model for the **true DG**
- A Turing model for the **user-defined model**
- A number of samples **N**
- (An **analytic posterior**)

Do the examples include an example where the joint posterior is exactly symmetric?
If not point that out and point to Betancourt case study to show how to handle 
those cases.

## Stacking for Multimodal Posteriors

1. Run MCMC for multiple Chains
2. (Optional:) Detect clusters in chains
3. Determine stacking weights by optimising the weights according to predictive 
    performance weights
4. Check convergence. We might have missed some modes. Ideally, we would check 
    by monitoring performance on some hold-out dataset.

## Key takeaways

- Multimodality is often a sign that we need to improve our model
- Either we have too little data which might be fine
- Or our model is misspecified and we need to expand it

- Part 2: What about neural nets?

## Conclusions

Authors recommend stacking should be used in the model exploration phase because 
it allows to run shorter chains and diagnose potential issues with the model.

Running multiple chains in parallel is (almost) free on modern hardware so this 
stacking approach is a promising avenue to quickly diagnose misspecified models 
and further use it in prediction task when we assume that our model is misspecified.

This should be able to be used in cases when model is underspecified. However, in 
that case we should be able to know beforehand that our model is underspecified.

Open research question:
- How do we make sure different chains explore a *diverse set of solutions*?

If the Bayes posterior is suboptimal for prediction under model misspecification 
does that mean Bayes is useless? Not necessarily. Instead multimodality and bad 
predictions could show us that our model needs to be expanded/improved. The solution 
then might not lie in finding a better inference/learning algorithm but instead 
find a principled way to expand our models.