# ece49595nl-GloVe
This repository is purely focused on implementation

## Requirements

Boost

## Usage

To train-

```
mkdir -p build
cmake ..
cmake --build . --target=glove
```

The build system will load values from the config file and build the requested target.

The targets `gen_frequencies`, `create_cooccurrences`, and `glove` can be specified during 
the build.  Each target depends on the previous one and only the furthest target needs to 
be specified.  The default value is `glove`.

## Procedure

1. Preprocess dataset into space-separated list of tokens
2. Generate vocab list with frequencies
3. Generate co-occurrences file
4. Train model

## Training details

### Cost function

$`
J = \sum\limits_{i,j=1}^{V} 
f(X_{i,j})
(w_i^T\tilde{w}_j + b_i + \tilde{b}_j - logX_{i,j})^2
`$

With a vector length of 3, this would be

$`
J = \sum\limits_{i,j=1}^{V} 
f(X_{i,j}) (
w_{i1}\tilde{w}_{j1} + w_{i2}\tilde{w}_{j2} + w_{i3}\tilde{w}_{j3}
+b_i-\tilde{b}_i-logX_{i,j}
)^2
`$

### Function f

$f$ is defined as a function of $X_{i,j}$ that weights frequencies in order to limit 
weighting of extremely frequent words.  

According to the paper

>1. f (0) = 0. If f is viewed as a continuous
function, it should vanish as x → 0 fast
enough that the limx→0 f (x) log2 x is finite.
>2. f (x) should be non-decreasing so that rare
co-occurrences are not overweighted.
>3. f (x) should be relatively small for large val-
ues of x, so that frequent co-occurrences are
not overweighted.

Also according to the paper [1](#ref1), one working function is 

$f(x) =
\begin{cases} 
   (x / x_{max})^\alpha & \text{if } x < x_{max} \\
   1 & \text{otherwise}
  \end{cases}
$

Given a known $x_{max}$ and $0 < \alpha < 1$.

Our implementation will use this suggested function to begin with.

### Training equations

We will attempt an implementation of the Hogwild! algorithm described in [3.](#sources).

We first identify that our cost function matches the form 
$`f(x) = \sum\limits_{e \in E} f_e (x_e)`$
where $E$ is the set $`\{i,j \in \N \mid X_{i,j} > 0\}`$.

In order to perform the algorithm, we sample $`e \in E`$ randomly and reevaluate $w_i$, 
$\tilde{w}_j$, $b_i$, and $\tilde{b}_j$ as $x_v \leftarrow \gamma G_{ev}(x_e)$ where 
$x_v$ is each term in our vectors and $\gamma$ is our learning rate.  In the GloVe 
cost function, $G_e(x_e)$ is the partial derivative of $J$ with respect to each term.

$`w_i \leftarrow w_i - \gamma \frac{\partial J}{\partial w_i} = 
w_i - \gamma 2f(X_{ij})(w_i^T\tilde{w}_j + b_i + \tilde{b}_j - log(X_{ij}))\tilde{w}_j`$

$`\tilde{w}_j \leftarrow \tilde{w}_j - \gamma \frac{\partial J}{\partial \tilde{w}_j} = 
\tilde{w}_j - \gamma 2f(X_{ij})(w_i^T\tilde{w}_j + b_i + \tilde{b}_j - log(X_{ij}))w_i`$

$`b_i \leftarrow b_i - \gamma \frac{\partial J}{\partial b_i} = 
b_i - \gamma 2f(X_{ij})(w_i^T\tilde{w}_j + b_i + \tilde{b}_j - log(X_{ij}))`$

$`b_j \leftarrow b_j - \gamma \frac{\partial J}{\partial b_j} = 
b_j - \gamma 2f(X_{ij})(w_i^T\tilde{w}_j + b_i + \tilde{b}_j - log(X_{ij}))`$

The algorithm also requires Lipschitz continuous differentiability of $f$, which we have
assuming that our function $f$ inside the cost function is differentiable, and given that
$X_{ij}$ is bounded.  It also requires that $f$ is strongly convex, which we cannot
guaruntee given the non-linearity of the $f$ in the cost function.  This method
may not reach a global minimum, but our terms should descend along the gradient regardless.

### Learning rate adjustment

## Implementation

### Tokenization

### Frequency generation

The frequencies are generated by reading in the file one word at a time using `std::ifstream` 
and counting frequencies using `std::unordered_map`.  This is written to an output text file 
by space separated pairs.

### Co-occurrence file generation

The cooccurrences are found by iterating through the corpus and updating the lookup table 
on the indices of the words.  The window size is defined at compile time, as are the sizes of 
the vocabulary and corpus.  We then shuffle and write the coocurrences to a binary file.  As 
the cost function relies on all i,j pairs, only i,j pairs with relevant cooccurrences need to 
be considered.  Thus, our training only needs to iterate through each cooccurrence set, allowing 
us to shuffle all cooccurrences similar to the paper's original implementation.

### Training

For the asynchronous stochastic gradient descent method, our updates are required to be atomic, 
evenly distributed, and random.  As our $i,j$ pairs are independent, we can flatten them and
randomize each cooccurrence frequency, which is done in the cooccurrence generation step.

## Sources

[GloVe paper](https://aclanthology.org/D14-1162.pdf)