
# TS-SHAP Package

## Introduction

TG-SHAP (TimeGroup-SHAPley) is a novel method designed to provide interpretability for Machine Learning (ML) and Deep Learning (DL) models that work with time series data. This method is particularly useful for models that perform Anomaly Detection (AD), allowing operators to understand the influence of different time instants on the model's predictions.

## Shapley Values

The Shapley value method, rooted in cooperative game theory, is a model-agnostic approach that assigns a specific contribution to each input feature for individual model predictions. Despite its robustness and comprehensibility, calculating Shapley values for models with large input spaces is computationally intensive. This complexity increases for models that exploit data sequentiality, common in AD models.

### Mathematical background

In cooperative game theory, the Shapley value is a way to distribute the total gains generated by a coalition of players. THe Shapley value for a player $i$ in a cooperative game with a set of players $N$ is given by:

$\varphi_i(v) = \sum_{S \subseteq N \setminus \{ i\}} \frac{|S|! \cdot (|N| - |S| -1)!}{|N|!}  ( v(S \cup i) - v(S)  )$

where: 
  + $v$ is the value function that assigns a real number to each subset $S$ of players, representing the total payoff of that subset.
  + $N$ is the set of all players.
  + $S$ is a subset of $N$ that does not include player $i$.
  + $|S|$ is the number of players in S.
  + $|N|$ is the total number of players.

This equation accounts for all possible coalitions $S$ that player $i$ can join, weighing the marginal contribution of $i$ to each coalition by the likelihood of that coalition forming.

### Application to ML/DL models

In order to adapt this method for interpretability, several steps should be taken into account:
  + **Identifiying the players**: Each input feature of the model $i$ is considered a player in the cooperative game.
  + **Defining the value function**: For every point $x_*$ and every subset of features $S \subseteq N$, the value function $v(S)$ is tipically defined as the expected model output when only the features in $S$ are known. Formally, $v(S) = \mathbb{E}(f(X)|X^S)$ where $X^S$ represents the values of the features in $S$, and $f$ is the model's prediction function.
  + **Computing the Shapley value**: For each feature $i$ in a point $x_*$ the Shapley value is computed using the formula:

  $\varphi_i(v, x_{\*}) = \sum_{S \subseteq N \setminus \{ i\}} \frac{|S|! \cdot (|N| - |S| -1)!}{|N|!}  ( v(S \cup i, x_{\*}) - v(S, x_{\*})  )$


## TG-SHAP

TG-SHAP addresses these challenges by grouping features by time instants, significantly reducing the computational complexity from $2^{p \cdot w}$ to $2^w$, where $p$ is the number of features measured at each instant and $w$ is the number of time instants in a window. This transformation maintains the interpretability of the model's decisions, helping identify which time instants most influence the model's predictions.

This interpretability method developed is inspired by the theory presented in [GroupShapley](
https://doi.org/10.48550/arXiv.2106.12228) , where the authors propose grouping the input features of ML/DL models to calculate the Shapley value for each group instead of calculating it for each individual feature. Furthermore, they demonstrate that by making this grouping, the axioms that justified the original adaptation of the method to the field of interpretability are still preserved. Based on this, TG-SHAP is proposed, a novel way to apply GroupShapley to interpret ML/DL models that handle time series.

Let $x$ denote a time window, consisting of $p$ features measured over $w$ time instants. To refer to the value of a feature $j$ at a time instant $i$, we denote it as $x_{t_i,j}$. To denote the vector of all features at a certain instant $i$, we use $x^{t_i}$, and to refer to only a subset of these $S \subseteq \{0, \ldots, p - 1\}$, we write $x^{t_i,S}$. The feature space, of size $w \times p$, is denoted as $N = \{(t_0, 0), \ldots, (t_0, p - 1), \ldots, (t_{w-1}, 0), \ldots, (t_{w-1}, p - 1)\}$, whose number of subsets is $|P(N)| = 2^{w \times p}$.

The first step is to decide the criterion for grouping features. In this case, it was decided to form a group for each time instant of the model's input window.

![Agrupación de características](https://github.com/ManuelFranco/TG-SHAP/assets/81265002/646f2ab6-1d3b-4a93-81a4-f19cf8a836a4)

Next, for each window $x_{∗}$ and each coalition $S$ of time instants, a value function $v(x_{∗}, S)$ must be determined. This function is theoretically defined as the expected value of the model conditioned on the time instants $S$ in the window $x_{∗}$ (denoted as $x_{∗}^S$), that is, $v(S, x_{∗}) = \mathbb{E}(f(x)|x^S = x_{∗}^S)$. In practice, this calculation is unfeasible because it involves knowing the probability distribution of the model $f(\cdot)$. To resolve this, it is decided to directly estimate the value function.

To perform the estimation, a set of a certain size $K$ is created, which we will call support, and it will be formed by a representative subset of the windows used to train the model. Then, for each window $x_{∗}$ and each subset $S$, $K$ windows are created formed by the combination of the instants $S$ from $x_{∗}$ and the $\bar{S}$ (the instants that are not in $S$) from each of the $K$ windows in the support set. Once this is done, the model is evaluated on each of these windows, obtaining the probability of belonging to the class originally predicted by the model. The value function for the coalition over the window is defined as the average of the probabilities of belonging to this class obtained for each of these combinations. Thus, the value of the coalition $S$ in the window $x_{∗}$ is approximated as $v(S, x_{∗}) \approx \frac{1}{K} \cdot \sum (f(x_{k}^{\bar{S}}, x_{∗}^S))$.
![soporte](https://github.com/ManuelFranco/TG-SHAP/assets/81265002/937a8b2a-deb3-488e-870f-e579b16d52a4)

![pipeline](https://github.com/ManuelFranco/TG-SHAP/assets/81265002/96830c0a-5995-4ac6-8200-c25b34bcba5f)

## Usage

To use the `tg_shap` function provided, you need to have a trained ML/DL model, a support dataset, and a test dataset. Here are the steps to use the TG-SHAP code effectively:

1. **Prepare the Datasets**: Ensure you have a support dataset and a test dataset formatted correctly. The support dataset should be representative of the data used to train the model.

2. **Set up the Model**: Ensure your model is trained and ready to be evaluated. The model should be compatible with the input data format.

3. **Define the Window Size**: Set the window size parameter based on the time series data you are working with.

4. **Run the TG-SHAP Function**: Call the `tg_shap` function with the necessary parameters:
   - `MODEL`: The trained model to be interpreted.
   - `SupportDataset`: The dataset used for support, providing context for the Shapley value calculations.
   - `TestDataset`: The dataset on which the interpretability analysis will be performed.
   - `windowSize`: The size of the time window for the time series data.

5. **Analyze the Results**: The function returns several outputs:
    + `sumas`: A tensor containing the sum of Shapley values for each instance in the test dataset.
    + `shapley_values`: A tensor containing the Shapley values for each time instant in each instance.
    + `diccionario_shap_aciertos`: A dictionary indicating whether the predicted class matches the actual class for each instance.
    + `df`: A DataFrame with detailed results including Shapley values, predicted probabilities, and accuracy information.

By following these steps, you can utilize the TG-SHAP method to gain insights into the interpretability of your ML/DL model's predictions on time series data. An example of use can be found [here](ExampleOfUse.ipynb)


## Installation

Before you can install and use TG-SHAP, ensure that you have the following prerequisites:

1. **Python**: TG-SHAP requires Python 3.6 or later. You can download Python from the [official website](https://www.python.org/downloads/).
2. **pip**: This is the package installer for Python. It is usually included with Python, but you can install or upgrade it using the following command:

   ```sh
   python -m ensurepip --upgrade
   ```

To install TG-SHAP, follow these steps:

1. **Clone the Repository**: First, clone the TG-SHAP repository from GitHub. Open your terminal or command prompt and run:
   ```sh
   git clone https://github.com/ManuelFranco/TG-SHAP.git
   ```

2. **Navigate to the Directory**: Change to the TG-SHAP directory:
   ```sh
   cd TG-SHAP
   ```

3. **Install Dependencies**: Install the required dependencies using pip. The required packages are listed in the \`requirements.txt\` file. Run:
   ```sh
   pip install -r requirements.txt
   ```

4. **Install TG-SHAP**: Finally, install TG-SHAP:
   ```sh
   pip install .
   ```

## Verification

To verify that TG-SHAP has been installed correctly, you can run a simple test. Open a Python interpreter and try importing the package:
```python
import tg_shap
print("TG-SHAP installed successfully!")
```

If there are no errors, the installation was successful.


## License

MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
## Acknowledgements

This project is based on the work presented in the master's thesis by Manuel Franco de la Peña, Universidad de Murcia, 2024.
