---
layout: article
title: The Intuition Behind Gradient Boosting & XGBoost
---

{% newthought 'In this article' %}, we present a very influential and powerful algorithm called *Extreme Gradient Boosting* or XGBoost <a href="#ref_one">[1]</a>. It is an implementation of Gradient Boosting machines which exploits various optimizations to train powerful predictive models very quickly. 

As such, we will first explain <a href="#gradient_boosting"><em>Gradient Boosting</em></a> <a href="#ref_two">[2]</a> to set readers in context. Then, we walk through the workings of <a href="#XGBoost">XGBoost</a> qualitatively, drawing connections to gradient boosting concepts as necessary. Finally, we talk about the various <a href="#optimizations">optimizations</a> implemented and the ideas behind them. 

In writing this article, I have made it a personal goal to be as qualitative as possible, bringing in equations only if it aids in the explanation. The goal is to provide readers with an intuition of how Gradient Boosting and XGBoost works. 

<h2 id="gradient_boosting">Gradient Boosting</h2>

Gradient Boosting involves building an ensemble of weak learners. It builds upon 2 key insights. Here's insight one.

>If we can account for our model's errors, we will be able to improve our model's performance.

<p id="example">
Let's use a simple example to back this idea. Let's say we have a regressive model which predicts 3 for a test case whose actual outcome is 1. If we know the error (2 in the given example), we can fine-tune the prediction by subtracting the error, 2,from the original prediction, 3 and obtain a more accurate prediction of 1. This begs the question, <em>"How do we know the error made by our model for any given input?"</em>, which leads us to our second insight.
</p>

>We can train a new predictor to predict the errors made by the original model.

Now, given any predictive model, we can improve its accuracy by first, training a new predictor{% sidenote 'sn-one' 'The predictor used to predict the error can be any function approximator.'%} to predict its current errors. Then, forming a new improved model whose output is the fine-tuned version of the original prediction. The improved model, which requires the outputs of both the *original predictor* and the *error predictor*, is now considered an ensemble of the two predictors. In gradient boosting, this is repeated arbitrary number of times to continually improve the model's accuracy. This repeated process forms the crux of gradient boosting.

### An Ensemble of Weak Learners

When training a new error-predicting model to predict a model's current errors, we regularize its complexity to prevent *overfitting*{% sidenote 'sn-two' 'A model which memorizes the errors for all of its training samples will have no use in the practical scenario.'%}. This regularized model will have *'errors'* when predicting the original model's *'errors'*. With reference to the <a href="#example">above example</a>, it might not necessarily predict 2. Since the new improved model's prediction depends on the new error-predicting model's prediction, it too will have errors albeit lower than before.

To mitigate this, we perform 2 measures. First, we reduce our reliance or trust on any single error predictor by applying a small weight, *$$ \eta $$* (typically between 0 to 0.1) to its output. Then, instead of stopping after 1 iteration of improvement, we repeat the process multiple times, learning new error predictors for newly formed improved models till the accuracy or error is satisfactory. This is summed up using the equations below where $$ x $$ is an input.

{% marginnote 'sn-three' 'Typically, the error-predicting model predicts the negative error and so, we use an addition instead of deduction.'%}

<p id="steps"></p>

$$
\begin{align*}
&improved\_model(x) = current\_model(x) + \eta \times error\_pred\_model(x)\\~\\
&current\_model(x) = improved\_model(x)\\~\\
&Repeat \ above \ 2 \ steps \ till \ satisfactory.
\end{align*}
$$

After every iteration, a new predictor accounting for the errors of the previous model will be learned and added into the ensemble. The number of iterations to perform and *$$ \eta $$* are hyperparameters. 

<br>

{% maincolumn 'assets/img/xgboost_1.png' 'If your idea of <em>Gradient Boosting</em> resembles the illustration, you are on the right track.' %}

### "Gradient" Boosting

Before ending off, we explore why this is called *"gradient"* boosting. It turns out that the error which we mentioned above is the gradient of the loss function $$ wrt $$ the model prediction and this is generalizable to any differentiable loss function. For example, when we differentiate the squared error loss function, $$ 0.5 (y_{true}-y_{pred})^2 $$, we get $$ y_{pred}-y_{true} $$ which uncoincidentally happens to be the "error" which we train our new error predictors on. Similarly, errors for other types of predictive problems such as classification problems can be expressed via the gradient. Since we are predicting the gradients, we call this gradient boosting. 

Mathematically, the derivative of the loss function, $$ \frac{\partial loss}{\partial pred} $$, gives the direction in which the predictions can be adjusted to maximize loss. In gradient boosting, we predict and adjust our predictions in the opposite (negative gradient) direction. This achieves the opposite (minimize the loss). Since, the loss of a model inversely relates to its performance and accuracy, doing so improves its performance.

Intuitively, we are shifting our model predictions in small steps towards directions{% sidenote 'sn-four' 'You can also think about this as a form of <em>Gradient Descend</em>.'%}  which improve the overall performance of our model. 


## XGBoost

XGBoost is a flavour of gradient boosting machines which uses Gradient Boosting Trees (gbtree) as the error predictor. It starts off with a simple predictor which predicts an arbitrary number (usually 0.5) regardless of the input. Needless to say, that predictor has a very high error rate. Then, the <a href="#steps">above idea</a> is applied till the error is brought to a minimum. In XGBoost, training of the error prediction model is not done by trivially optimizing the predictor on  $$ (feature, error) $$ pairs. Next, let's take a look at how they are built.

### Gradient Boosting Tree

In XGBoost, a gbtree is learnt such that the overall loss of the new model is minimized while keeping in mind not to *overfit* the model. Note that in this section, we are talking about 1 iteration of the above idea. To understand it better, let's start from the simplest possible tree which makes no split and predicts the same value regardless of the input. This tree is extremely simple, is independent of the input and is definitely underfitted. Nonetheless, it can still help in decreasing loss. The problem mentioned can be represented by the equations below. 

$$

~\\ ~ \\

Loss(o) = \min_{o}  \sum_{i = 1}^N loss(y_i, f(x_i)+o) + \frac{1}{2}\lambda o^2 \\ ~ \\

\begin{align*}
&where\ N\ is\ the\ number\ of\ samples,\ f\ is\ the\ original\ model,\\
&\lambda \ is\ the\ L2\ regularization\ parameter\ and\ o\ is\ the\ value\ which\ we\ want\ to\ find.
\end{align*}

~ \\ ~ \\ 

$$

The L2 regularization{% sidenote 'sn-five' 'Refer to this <a href="https://towardsdatascience.com/intuitions-on-l1-and-l2-regularisation-235f2db4c261">article</a> for an explanation about L2 regularization.'%} applied, as represented by the $$ \frac{1}{2}\lambda o^2 $$ term, has been shown experimentally to be effective in preventing overfitting. While not useful in this already underfitted model, it will come into relevance as we increase the tree complexity. A problem like this can be solved by differentiating the expression $$ wrt\ o $$, setting the derivative to 0 and then finding the corresponding $$ o $$. Unfortunately, the expression we see above is hard to differentiate. To get around this, we approximate that expression with simpler terms using Quadratic Approximation{% sidenote 'sn-five' 'You can watch this <a href="https://www.khanacademy.org/math/multivariable-calculus/applications-of-multivariable-derivatives#quadratic-approximations">Khan Academy series</a> to understand Quadratic Approximation and do the derivation as an exercise.'%}. 

$$

~ \\ ~ \\ 

Loss(o) \approx  \sum_{i = 1}^N [loss(y_i, f(x_i)) + \frac{\partial loss}{\partial \widehat{y}}(y_i, f(x_i))o + \frac{1}{2} \frac{\partial^2 loss}{\partial \widehat{y}^2}(y_i, f(x_i)) o^2] + \frac{1}{2}\lambda o^2 \\ ~ \\

\approx \sum_{i = 1}^N [loss(y_i, f(x_i)) + g_io + \frac{1}{2} h_i o^2] + \frac{1}{2}\lambda o^2 \\~ \\

where\ g_i = \frac{\partial loss}{\partial \widehat{y}}(y_i, f(x_i)) \ and\ h_i=\frac{\partial^2 loss}{\partial \widehat{y}^2}(y_i, f(x_i)).

~ \\ ~ \\ 

$$

This simplified expression can be differentiated easily and after setting the derivative to 0, we can solve and obtain $$ o $$. It turns out that $$ o $$ is the following. 

{% marginfigure 'mf-1' 'assets/img/Figure 2.png' 'An illustration of the scenario given: We find a single best adjustment $$ o $$ which we can apply to any sample in our dataset to minimize overall loss.' %}

$$

~ \\ ~ \\ 

o = \frac{\sum_{i = 1}^N g_i}{\sum_{i = 1}^N h_i + \lambda}

~ \\ ~ \\ 

$$

Keep in mind that, now, given a model $$ f $$ and a set of samples, we can find a single adjustment $$ o $$ which best improves our model. <a id="note_1"></a>*Note that $$ o $$ can also be substituted back into the equation to compute the value of $$ Loss $$.* The remaining of this section talks about how we can further improve (decrease the loss) by increasing the complexity of our simple model (growing the tree). Here's the overall idea. 

>  By cleverly dividing the samples into subgroups and then finding $$ o $$ for each subgroup (using the method above), the performance of the model can be further improved (loss can be brought lower).

The samples can be divided using split conditions. For example, if a split condition is *"feature x less than 10"*, samples whose feature x has value less than 10 will go into 1 subgroup and the rest, to the other group. Each subgroup can be further divided iteratively if necessary. These splits divide the original feature space into smaller subspaces and samples in each subspace form a subgroup. For each subgroup, its optimal $$ o $$ and Loss{% sidenote 'sn-six' '<a href="#note_1">Remember</a> that the minimum loss for a subgroup can be computed by substituting optimal $$ o $$ back into $$ Loss $$.'%} can be solved using the above technique. The overall loss, $$ Loss $$, is the summation of the loss of each subgroup (leaves in the decision tree).{% marginfigure 'mf-2' 'assets/img/Figure 3.png' 'An illustration of the discussed concept: In this example, the features space is divided into 3 segments with splits $$ B < 2 $$ and $$ A < 2.5 $$. The optimal $$ o $$ for each subgroup is then computed using the discussed technique.' %}

At each group or subgroup, the decision of whether to split and if so, which split to use depends on whether a split can reduce the loss of that group and how much each split decreases loss. We always choose the split which minimizes $$ Loss $$ and will not split if $$ Loss $$ cannot be decreased. 

Let's describe what's happening intuitively. The current model has different levels of errors in different parts of the feature space. It overpredicts for some samples, underpredicts for others and by varying magnitudes. *By segmenting the feature space such that the errors in each subgroup are similar, more specific and thus, better adjustments can be computed for each subgroup, enhancing overall model performance.*

### Overfitting

To prevent *overfitting*, several measures are implemented. We discuss two important ones here. First, the maximum height of trees grown can be capped by the user. This helps in limitings the number of subgroups (leaves) which can be formed. Second, the decrease in loss from a split must exceed a certain threshold set by the user for XGBoost to allow it. This is modeled into the $$ Loss $$ via an additional regularization term, $$ \gamma T\ where\ T\ is\ the\ number\ of\ leaves $$. This was omited earlier on to prevent confusion. 


## Optimizations

Here are interesting optimizations used by XGBoost to increase training speed and accuracy.

**Weighted Quantile Sketch** for finding approximate best split - Before finding the best split, we form a histogram for each feature. The boundaries of the histogram bins are then used as candidate points for finding the best split. In the Weighted Quantile Sketch, the data points are assigned weights based on the "confidence" of their current predictions and the histograms are built such that each bin has approximately the same total weight (as opposed to the same number of points in the traditional quantile sketch). As a result, more candidate points and thus, a more detailed search will exist in areas where the model is doing poorly.

**Parallelization** for faster tree building process - When finding optimal splits, the trying of candidate points can be parallelized at the feature/column level. For example, core 1 can be finding the best split point and its corresponding loss for *feature A* while core 2 can be doing the same for *feature B*. In the end, we compare the losses and use the best one as the split point.

**Sparsity-Aware Split Finding** for handling sparse data - XGBoost handles this sparsity, which may result from missing values or frequent zero entries from one-hot encodings by assigning them a default direction at every node in the tree. The default direction is chosen based on which reduces the $$ Loss $$ more. On top of this, XGBoost ensures that sparse data are not iterated over during the split finding process, preventing unneccessary computation.

**Hardware Optimizations** - XGBoost stores the frequently used $$ g_i $$s and $$ h_i $$s in the cache to minimize data access costs. When disk usage is required (due to data not fitting into memory), the data is compressed before storage, reducing the IO cost involved at the expense of some compression computation. If multiple disks exist, the data can be <a href="https://en.wikipedia.org/wiki/Shard_(database_architecture)">sharded</a> to increase disk reading throughput.

**Column and Row Subsampling** - To reduce training time, XGBoost provides the option of training every tree with only a randomly sampled subset of the original data rows where the size of this subset is determined by the user. The same applies to the columns/features of the dataset. Apart from savings in training time, subsampling the columns during training has the effect of decorrelating the trees which can reduce overfitting and boost model performance. This idea is also used in the Random Forest{% sidenote 'sn-seven' '<a href="https://en.wikipedia.org/wiki/Random_forest">https://en.wikipedia.org/wiki/Random_forest</a>'%} algorithm. 

##  End Note and References

Cheers, we have reached the end. Hopefully, this has helped you. Feel free to E-mail me (liangweitan300895@gmail.com) for feedback, questions or even a chat.

[1]<cite id="ref_one">[T. Chen and C. Guestrin, "XGBoost: A Scalable Tree Boosting System," no. arXiv:1603.02754 [cs.LG], 2016](https://arxiv.org/abs/1603.02754)</cite>
<br>
[2]<cite id="ref_two">[J. H. Friedman, "Stochastic Gradient Boosting," 1999](https://statweb.stanford.edu/~jhf/ftp/stobst.pdf)</cite>





