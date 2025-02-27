# Linear Regression

- Linear Model = Weighted sum of input features + Bias (also called intercept).

  $\hat{y} = \theta_0 + \theta_1 x_1 + \theta_2 x_2 + \dots + \theta_n x_n$

  where -
  - $\hat{y}$ is the predicted value
  - $n$ is the number of features
  - $x_i$ is the $i^{th}$ feature
  - $\theta_j$ is the $j^{th}$ model parameter, including the bias term $\theta_0$ and the feature weights $\theta_1, \theta_2, \dots, \theta_n$

- Vectorized form -

  $\hat{y} = h_{\theta}(x) = \theta^T X$

  where -
  - $h_{\theta}$ is the hypothesis function, using the model parameter $\theta$
  - $\theta$ is the model's parameter vector, containing the bias term $\theta_0$ and the feature weights $\theta_1, \theta_2, \dots, \theta_n$
  - $X$ is the instance’s feature vector, containing $x_0$ to $x_n$, with $x_0$ always equal to $1$
  - $\theta \cdot X$ is the dot product of the vectors $\theta$ and $X$, which is equal to $\theta_0 x_0 + \theta_1 x_1 + \theta_2 x_2 + \dots + \theta_n x_n$
  - Vectors are often represented as column vectors, therefore, we transpose the $\theta$ vector

> [!NOTE]
> Learning algorithms will often optimize a different loss function during training than the performance measure used to evaluate the final model. We choose the training loss function that is easy to optimize (e.g., log loss), and performance metrics that are closer to business needs (e.g., precision/recall). However, a training loss is strongly correlated to the metric.

- Root Mean Square Error (RMSE) is the most common performance measure of a regression model, therefore we need to find the value of $\theta$ that minimizes the RMSE.

  $MSE = \frac{1}{n} \sum (y_i - \hat{y}_i)^2$
