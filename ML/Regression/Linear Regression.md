# Linear Regression

- Linear Model = Weighted sum of input features + Bias (also called intercept).

  **y_pred = θ<sub>0</sub> + θ<sub>1</sub> x<sub>1</sub> + θ<sub>2</sub> x<sub>2</sub> + ... + θ<sub>n</sub> x<sub>n</sub>**

  where -
  - **y_pred** is predicted value
  - **n** is the number of features
  - **x<sub>i</sub>** is the i<sup>th</sup> feature
  - **θ<sub>j</sub>** is the j<sup>th</sup> model parameter, including the bias tern **θ<sub>0</sub>** and the feature weights **θ<sub>1</sub>, θ<sub>2</sub>, ... , θ<sub>n</sub>**

- Vectorized form -

  **y_pred = h<sub>θ</sub>(x) = θ<sup>T</sup>.X**

  where -
  - **h<sub>θ</sub>** is the hypothesis function, using the model parameter θ
  - θ is the model's parameter vector, containing the bias term **θ<sub>0</sub>** and the feature weights **θ<sub>1</sub>, θ<sub>2</sub>, ... , θ<sub>n</sub>**
  - **X** is the instance’s feature vector, containing **x<sub>0</sub>** to **x<sub>n</sub>**, with **x<sub>0</sub>** always equal to _1_
  - **θ.X** is the dot product of the vectors θ and X, which is equal to **θ<sub>0</sub>x<sub>0</sub> + θ<sub>1</sub> x<sub>1</sub> + θ<sub>2</sub> x<sub>2</sub> + ... + θ<sub>n</sub> x<sub>n</sub>**
  - Vectors are often represented as column vectors, therefore, we transpose the **θ** vector

> [!NOTE]
> Learning algorithms will often optimize a different loss function during training than the performance measure used to evaluate the final model. We choose the training loss function that is easy to optimize (eg - log loss), and performance metrics which is closer to business (eg - precision/recall). However, a training loss is strongly correlated to the metric.

- Root Mean Square Error (RMSE) is the most common performance measure of a regression model, therefore we need to find the value of θ that minimizes the RMSE.

  **$MSE = \frac{1}{n} \sum (y_i - \hat{y}_i)^2$**



