@def title = "MCMC Diagnostics"
@def published = "1 January 2021"
@def tags = ["stats", "mcmc"]
@def hasmath = true

# MCMC Diagnostics

MCMC gives asymptotic guarantees for producing samples from a target distribution.
However, we do not have time to wait until forever and only run MCMC chains for 
a finite amount of time. Therefore we need diagnostics to tell us how reliable 
our estimates based on the MCMC samples are. Modern computers allow us to run 
MCMC on very high-dimensional problems which makes it infeasible to solely rely 
on visual diagnostics such as traceplots (although they are still important).

General MCMC diagnostics:
- $\hat{R}$
    - This should only be trusted if the ESS > 400 because it depends on estimates
        of the variance and autocorrelation
    - We are finding two estimators for $var(\theta | y)$ which ideally should 
        agree with each other but one estimator consistently overestimates and 
        the other consistently underestimates. By taking the ratio of the two 
        we get a sense of the lack of convergence.
- the effective sample size (ESS)
- Monte Carlo Squared Error (MCSE)
- Bulk- and tail-ESS

ESS and MCSE estimates are for evaluating the posterior mean only! But can 
be adapted 

Open questions:
- What is r-hat on a high level?
- How do we calculate ESS and MCSE?

Dynamic HMC specific diagnostics:
- Estimated Bayesian fraction of missing information
- Divergences
    - Link to divergences case studies of betancourt

Resources to link to:
- Stan warning page
- Bayesplot vignettes
- Vehtari paper
- Betancourt paper

## Open questions

- What are the "default" plots one should do after running inference? ArViZ and 
    bayesplot provide a gazillion different plotting options so it is a bit 
    overwhelming to understand what we are looking for
    - Can the advice be worked into a decision tree which can potentially be 
        automated?
    - I guess partly it depends on what problem you are trying to solve and 
        what parameters one is interested in.