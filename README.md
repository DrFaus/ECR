# 3. Proposed Exam Copying Room Algorithm

The following methodology describes the **Exam Copying Room (ECR)** algorithm implemented in the reference MATLAB code. Mathematical expressions use GitHub-compatible LaTeX notation.

## 3.1. General description

The Exam Copying Room (ECR) algorithm is a population-based stochastic optimization method inspired by the social interactions that may occur among students during an examination. The underlying metaphor considers a group of students attempting to improve their examination scores by selectively imitating the answers of nearby classmates. Each student represents a candidate solution, while each examination question corresponds to a decision variable of the optimization problem.

Unlike imitation strategies in which an individual follows a single reference solution, ECR allows each decision variable to be influenced by a potentially different individual. This mechanism introduces coordinate-wise information exchange, enabling candidate solutions to combine information obtained from multiple members of the population.

The algorithm incorporates a circular neighborhood topology, probabilistic social selection, entropy-based adaptation of social temperature, individual failure tolerances, a logistic imitation policy, and stochastic perturbations. These mechanisms jointly regulate how individuals select sources of information, determine the intensity of imitation, and explore alternative solutions.

The optimization process follows an iterative structure in which candidate solutions are generated through social imitation and stochastic exploration. A greedy selection criterion preserves improving solutions, while unsuccessful attempts influence subsequent imitation behavior.

The following subsections describe the mathematical formulation and implementation of ECR.

## 3.2. Optimization problem and population initialization

Consider a continuous, bounded optimization problem expressed as

$$
\min_{\mathbf{x}\in\Omega}f(\mathbf{x}), \tag{1}
$$

where $f:\mathbb{R}^{D}\rightarrow\mathbb{R}$ is the objective function, $D$ is the number of decision variables, and the feasible search domain is defined by

$$
\Omega=\prod_{j=1}^{D}[L_j,U_j]. \tag{2}
$$

Here, $L_j$ and $U_j$ denote the lower and upper bounds of the $j$-th decision variable, respectively.

ECR maintains a population of $N$ individuals, referred to as students or examinations,

$$
X=\{\mathbf{x}_1,\mathbf{x}_2,\ldots,\mathbf{x}_N\}, \tag{3}
$$

where each individual is represented by a vector

$$
\mathbf{x}_i=[x_{i1},x_{i2},\ldots,x_{iD}]^{T}. \tag{4}
$$

The initial population is generated using a uniform random distribution over the feasible search domain:

$$
x_{ij}^{(0)}=L_j+r_{ij}(U_j-L_j), \tag{5}
$$

where $r_{ij}\sim\mathcal{U}(0,1)$.

After initialization, the objective function is evaluated for every individual:

$$
F_i^{(0)}=f(\mathbf{x}_i^{(0)}). \tag{6}
$$

The individual with the lowest objective function value is identified, and its solution and fitness are stored as the global best-so-far solution:

$$
\mathbf{x}^{*}=\underset{\mathbf{x}_i\in X}{\operatorname{argmin}}\,f(\mathbf{x}_i). \tag{7}
$$

Additionally, each individual is assigned a failure counter $c_i=0$ and an individual failure tolerance $\tau_i$, randomly generated as

$$
\tau_i\sim\mathcal{U}_{\mathrm{int}}(5,15). \tag{8}
$$

These tolerances introduce heterogeneity in the population, allowing different individuals to respond differently to repeated unsuccessful attempts.

## 3.3. Social neighborhood and probabilistic selection

ECR employs a circular neighborhood topology to define the social interactions among individuals. Each student is connected to three neighboring individuals on either side of its position in the population.

The neighborhood of individual $i$ is defined as

$$
\mathcal{V}_i=\left\{\left((i+s-1)\bmod N\right)+1:\ s\in\{-3,-2,-1,1,2,3\}\right\}, \tag{9}
$$

where the modular operation ensures circular connectivity. The population size must be at least seven, as required by the reference implementation.

Consequently, each individual has six immediate social connections. This arrangement establishes a local information-sharing structure rather than allowing unrestricted interaction with the entire population.

The *list of candidate sources* depends on the number of consecutive unsuccessful attempts:

$$
\mathcal{S}_i=\begin{cases}
\mathcal{V}_i\mathbin{\|}\{i\}, & c_i\leq\tau_i,\\[4pt]
\mathcal{V}_i\mathbin{\|}\{b\}, & c_i>\tau_i,
\end{cases} \tag{10}
$$

where $b$ is the index of the best-ranked individual identified at the most recent population-level update and $\|$ denotes concatenation of candidate entries.

Under normal conditions, an individual may select its own solution as a source of information. However, after exceeding its failure tolerance, this self-reference is replaced by the best-ranked individual. The candidate entries are retained even when the best-ranked individual is already present in the neighborhood; therefore, repeated references are possible.

Rather than selecting the best available individual deterministically, ECR assigns a probability to each candidate entry using a softmax function:

$$
p_{i\ell}=\frac{\exp\left[-(F_{s_{i\ell}}-F_{\min,i})/T_s\right]}{\displaystyle\sum_{m=1}^{M}\exp\left[-(F_{s_{im}}-F_{\min,i})/T_s\right]}, \tag{11}
$$

where $s_{i\ell}$ denotes the individual referenced by candidate entry $\ell$, $M=7$ is the number of candidate entries, $F_{\min,i}$ is the minimum fitness among these entries, and $T_s$ is the social temperature. Subtracting $F_{\min,i}$ provides numerical stabilization without changing the resulting probabilities.

This probabilistic selection mechanism favors individuals with lower objective function values while preserving the possibility of selecting other available information sources. The social temperature controls the concentration of the probability distribution. Lower values favor stronger preference for high-performing individuals, whereas higher values distribute the selection probabilities more uniformly.

## 3.4. Entropy-based adaptive social temperature

To regulate the stochasticity of social selection, ECR employs an adaptive social temperature based on Shannon entropy.

For individual $i$, the entropy associated with its probability distribution is calculated as

$$
H_i=-\sum_{\ell=1}^{M}p_{i\ell}\ln(p_{i\ell}), \tag{12}
$$

where the conventional value $0\ln 0=0$ is adopted.

The maximum entropy for a distribution containing $M$ candidate entries is

$$
H_{\max}=\ln(M). \tag{13}
$$

A normalized entropy measure is then obtained:

$$
\widehat{H}_i=\frac{H_i}{H_{\max}}. \tag{14}
$$

This quantity approaches zero when the probability distribution is highly concentrated and approaches one when the candidate entries have approximately equal selection probabilities.

The social temperature is updated according to

$$
T_s^{+}=\operatorname{clip}\left(T_s\left[1+\gamma(H_{\mathrm{target}}-\widehat{H}_i)\right],T_{\min},T_{\max}\right), \tag{15}
$$

where $\operatorname{clip}(z,a,b)=\min(b,\max(a,z))$.

The parameters used in the reference implementation are summarized below.

| Parameter | Value |
| :--- | ---: |
| Initial social temperature, $T_s^{(0)}$ | 0.9 |
| Minimum temperature, $T_{\min}$ | 0.4 |
| Maximum temperature, $T_{\max}$ | 2.5 |
| Entropy sensitivity, $\gamma$ | 0.6 |
| Target normalized entropy, $H_{\mathrm{target}}$ | 0.65 |

When the normalized entropy is below the target value, the temperature increases, reducing the concentration of the selection distribution in subsequent selections. Conversely, when entropy exceeds the target, the temperature decreases, encouraging stronger preference for high-performing individuals.

An important characteristic of this implementation is that the social temperature is a shared population-level variable, updated sequentially **after** the probability distribution of each individual has been calculated. Thus, the updated temperature influences subsequent selections, including those occurring within the same population iteration.

## 3.5. Logistic adaptation of the imitation factor

In addition to determining whom to imitate, ECR adaptively regulates how strongly an individual moves toward the selected information source.

This behavior is controlled by an imitation factor $\alpha_i$, which depends on the individual's accumulated consecutive failures relative to its failure tolerance.

First, a normalized failure measure is defined:

$$
q_i=\min\left(1,\frac{c_i}{\tau_i}\right). \tag{16}
$$

A logistic response is then calculated as

$$
g(q)=\frac{1}{1+\exp[-k(q-q_0)]}, \tag{17}
$$

where $k$ controls the steepness of the logistic transition and $q_0$ determines its midpoint.

To ensure that the imitation factor remains within prescribed limits, the logistic response is normalized:

$$
\widehat{g}(q_i)=\frac{g(q_i)-g(0)}{g(1)-g(0)}. \tag{18}
$$

Finally, the imitation factor is obtained as

$$
\alpha_i=\alpha_{\min}+(\alpha_{\max}-\alpha_{\min})\widehat{g}(q_i). \tag{19}
$$

The parameters used are as follows.

| Parameter | Value |
| :--- | ---: |
| Minimum imitation factor, $\alpha_{\min}$ | 0.1 |
| Maximum imitation factor, $\alpha_{\max}$ | 0.9 |
| Logistic midpoint, $q_0$ | 0.6 |
| Logistic steepness, $k$ | 8 |

This mechanism produces a nonlinear relationship between unsuccessful attempts and imitation intensity. Individuals with few failures exhibit relatively weak imitation and therefore retain a greater proportion of their current solutions. As unsuccessful attempts accumulate, the imitation factor increases, producing stronger movement toward the selected sources.

Once the failure counter reaches its individual tolerance, the imitation factor saturates at its maximum value. The logistic adaptation introduces a behavioral response to search stagnation without requiring a fixed imitation intensity throughout the optimization process.

## 3.6. Candidate generation through coordinate-wise imitation

For each individual $i$, ECR generates a new candidate solution by independently selecting an information source for each decision variable.

Let $k_{ij}$ denote the selected source for the $j$-th variable of individual $i$. This source is sampled according to the social probability distribution defined in Eq. (11):

$$
k_{ij}\sim\operatorname{Categorical}\left(\mathcal{S}_i,\mathbf{p}_i\right). \tag{20}
$$

Different coordinates may therefore be influenced by different individuals.

The basic imitation operator is expressed as

$$
y_{ij}=x_{ij}+\alpha_i(x_{k_{ij},j}-x_{ij})+\varepsilon\xi_{ij}(U_j-L_j), \tag{21}
$$

where $\xi_{ij}\sim\mathcal{N}(0,1)$ and $\varepsilon=0.001$.

The first term retains the individual's current solution, while the second introduces a directional movement toward the selected source. The third term represents a small Gaussian perturbation scaled to the range of the corresponding decision variable. This scaling avoids directly using the same absolute perturbation magnitude for variables with different numerical ranges.

### Additional stochastic exploration

An additional perturbation mechanism is applied when an individual selects a source other than itself.

For each coordinate, an independent Bernoulli variable is generated:

$$
B_{ij}\sim\operatorname{Bernoulli}\left(\frac{\alpha_i}{3}\right). \tag{22}
$$

The complete candidate-generation operator is

$$
\boxed{y_{ij}=x_{ij}+\alpha_i(x_{k_{ij},j}-x_{ij})+0.001\,\xi_{ij}(U_j-L_j)+I(k_{ij}\neq i)B_{ij}\eta_{ij}\frac{U_j-L_j}{5}} \tag{23}
$$

where $\eta_{ij}\sim\mathcal{N}(0,1)$ and $I(\cdot)$ is an indicator function. The additional random number is used only when the perturbation is activated in the implementation.

This additional perturbation occurs with probability $\alpha_i/3$, provided that the selected source is different from the current individual. Consequently, the logistic imitation factor regulates two related aspects of candidate generation: the magnitude of movement toward selected sources and the probability of introducing additional stochastic exploration.

The resulting operator combines directional imitation with local and occasionally larger stochastic displacements.

## 3.7. Boundary handling

Because stochastic perturbations may generate infeasible candidate solutions, ECR employs a boundary-handling procedure based on reflection.

For each decision variable, if the candidate violates its lower bound, the coordinate is reflected according to

$$
y_{ij}\leftarrow 2L_j-y_{ij}\qquad\text{if }y_{ij}<L_j. \tag{24}
$$

Subsequently, if the coordinate exceeds its upper bound, it is reflected using

$$
y_{ij}\leftarrow 2U_j-y_{ij}\qquad\text{if }y_{ij}>U_j. \tag{25}
$$

Finally, a clipping operation is applied:

$$
y_{ij}\leftarrow\min\left(U_j,\max(L_j,y_{ij})\right). \tag{26}
$$

This procedure first attempts to preserve part of the stochastic displacement through reflection. The final clipping step guarantees feasibility when large displacements cannot be fully corrected by the preceding reflections.

## 3.8. Greedy selection and failure memory

After boundary handling, the newly generated candidate is evaluated:

$$
F_i'=f(\mathbf{y}_i). \tag{27}
$$

A greedy selection rule determines whether the candidate replaces the current solution:

$$
\mathbf{x}_i^{+}=\begin{cases}
\mathbf{y}_i, & F_i'<F_i,\\
\mathbf{x}_i, & F_i'\geq F_i.
\end{cases} \tag{28}
$$

The corresponding objective function value is updated consistently.

When an improvement occurs, the individual's consecutive-failure counter is reset:

$$
c_i^{+}=0. \tag{29}
$$

Otherwise, the unsuccessful attempt is recorded:

$$
c_i^{+}=c_i+1. \tag{30}
$$

The failure counter therefore contains information about the individual's recent optimization history. This information affects subsequent iterations through two mechanisms: the logistic imitation factor and the composition of the individual's social candidate set.

Whenever an accepted candidate improves upon the best objective function value found so far, the global best-so-far solution is also updated.

The population is processed sequentially, so improvements accepted for earlier individuals are available when later individuals calculate their social selection probabilities. However, the index identifying the current best-ranked population member is recomputed only after all individuals have been processed in the iteration.

The combination of greedy replacement and failure-dependent behavioral adaptation allows ECR to retain successful solutions while changing its search behavior following unsuccessful attempts.

## 3.9. Termination criterion and computational effort

The optimization process continues for a predefined number of iterations, $T_{\max}$.

At the end of each iteration, the best-so-far objective function value is stored to construct the convergence history:

$$
C(t)=f(\mathbf{x}^{*}(t)),\qquad t=1,\ldots,T_{\max}. \tag{31}
$$

The algorithm returns the best solution found and its associated objective function value:

$$
\mathbf{x}^{*},\qquad f^{*}=f(\mathbf{x}^{*}). \tag{32}
$$

In the implemented version, every individual generates exactly one candidate solution and performs one objective function evaluation per iteration.

Since the initial population also requires $N$ evaluations, the total number of objective function evaluations is

$$
\boxed{N_{\mathrm{FE}}=N(1+T_{\max}).} \tag{33}
$$

This expression provides a direct basis for controlling the computational budget in experimental comparisons.

Overall, ECR combines localized probabilistic information exchange, failure-dependent imitation, entropy-based social adaptation, and stochastic exploration within a population-based optimization framework. Its defining feature is the coordinate-wise selection of information sources, through which different components of a candidate solution may be influenced by different individuals.

The relative contribution of these mechanisms, as well as their relationship with the structural characteristics of optimization problems, will be investigated through the experimental methodology described in the following section.
