
# Introduction  
This is a summer project I did in 2018 summer in Imperial College London. The main task is to simulate the following logistic stochastic delay differential equation. 

$dX=X( \alpha -\beta X_\tau )dt+\sigma X dW$

where $\alpha$ and $\beta$ are parameters of the deterministic delay logistic equation, and $X_\tau(t)=X(t-\tau)$.

We were also askd to simulate the system when a pullback in time is set, in order to find pullback attractors.

I am going to explain some of the algorithms I used in the scripts.

# How Noise Realisation Is Implemented When Pullback Exists

A pullback means we have  t taking negative values. For example, if pullback is 200, then the system starts at t = -200.  We want to make sure that the noise realisation should be the same at each time point, so we can adjust the value of pullback and explore pullback attractors.  

Thus, in order to allow pullback to vary without altering the noise realisation, we should not generate noise forward from the very initial time point (e.g. t = -200). Instead, we should generate **two** streams of noise, starting from t = 0. 

Stream A  runs forward from t = 0 to $+\infty$.

Stream B runs backward from t = 0 to $-\infty$.

**NOTICE** : we need two different seeds to generate those streams, otherwise there is a symmetry in noise.

This makes sure that once the random seed is fixed, we can adjust pullback and the final time as we wish without altering the noise.

# Stochastic Integration Schemes
The philosophy of numeric integration is to discretise time, and use summation to replace integration.
 
Two integration schemes are used for integrations. In most of the scripts, Euler-Maruyama method are used to save computing time. Heun's Method  is only used when stated in the title of the scripts. More sophisticated integration schemes like Runge-Kutta, requires fractional time step, which I found infeasible for stochastic delay differential equation. 

##  Euler-Maruyama

The Euler-Maruyama method is basically a stochastic version of the Euler's method for deterministic equation. Under Euler-Maruyama method, our equation becomes

$X(t+dt)-X(t)=X(t)\left[\alpha +\beta X_\tau(t)\right]dt+\sigma X(t)[W(t+dt)-W(t)]$

$W(t+dt)-W(t)$ follows a normal distribution with variance $dt$. Thus, $W(t+dt)-W(t)$ is realised by drawing a sample from the normal distribution with variance $dt$. In fact, in my implementation, a smaller times step called **tDelta** is set, and $dt=R*tDelta$, where $R$ is an integer. Now

$W(t+dt)-W(t)=\displaystyle\Sigma_{i=0}^{R-1}[W(t+(i+1)*tDelta)-W(t+i*tDelta)]$

##   Heun's Method

Heun's Method is supposed to be more accurate than Euler's Method for integrating the deterministic equation, but it is more time- consuming. The scheme for integrating the random variable is the same as Euler-Maruyama, as Heun's Method only improves evaluation of the deterministic gradient. 

For the sake of simplicity, suppose our equation is 

$dX=\phi(X(t)) dt+\theta(X(t))dW$

Then, under Heun's method, we first use the same technique as Euler's method to find the value of X(t+dt) by

$X(t+dt)=X(t)+\phi(X(t)) dt+\theta(X(t))dW$

However, this $X(t+dt)$ is only an intermediate value. The purpose is to use this to evaluate $\phi(X(t+dt))$ and then use the average gradient to evaluate $X(t+dt)$ again.

$X(t+dt)=X(t)+\frac{1}{2}[\phi(X(t))+\phi(X(t+dt))] dt+\theta(X(t))dW$

# Lyapunov spectrum

The Lyapunov specturm is very useful in determining types of attractors. Negative Lyapunov spectra mean **stable** attractors, and positive Lyapunov spectra mean **unstable** attractors, and Lyapunov spectra with both positive and negative values mean **strange** attractors. 

The dynamics we are interested in is 

$\Phi:C[I,\mathbb{R}]\rightarrow C[I,\mathbb{R}]\quad \text{where } I=[-\tau,0]$

We use finite points to represent $X_\tau \in C[I,\mathbb{R}]$.

For the purpose of more robust computation, we use $NHistory=\tau/dt$ points, which are evenly spaced in the time domain, to represent the function, where $dt$ is set to guarantee that $NHistory$ is an integer. Now the delay function is approximated by a $NHistory$-dimensional object.  Thus, this $NHistory$-dimensional object can be represented by a $NHistory$-dimensional vector, indicating the values the object takes at each time point. (i.e. when $t=\tau+i*dt\quad$, where $i$ is an integer and $0\leq i \leq NHistory-1$   ) 

Since we are dealing with $NHistory$ -dimension, we choose the canonical basis $\{e_1,e_2,\dots, e_{Nhistory}\}$.  Aligning them together gives us the identity matrix. To estimate the Lyapunov spectrum,  we are going to see how this canonical basis develops when time span is very large, under the linearisation of the of stochastic delay differential equation. Since this system is autonomous, the dynamics of the tangent equation can be written in the form 

$V(t)=e^{tA}V_0,\quad A\in\mathbb{R}^{NHistory\times NHistory}$

Thus starting with $V_o=I$, we have $V(t)=e^{tA}$. The Lyapunov spectrum is a vector :

$L=\lim(t\rightarrow \infty)\{\frac{1}{t}\log ( \lambda_V(t))\}$

where $\lambda_V(t)$ is the vector of all real parts of eigenvalues of $V(t)$. 

Working from first principles, the tangent equation is found to be

$dV=[V(\alpha-\beta X_\tau)-\beta V_\tau X] dt+\sigma V dW$

Now by implementing the integration scheme, we can find $V(t)$, and estimate the Lyapunov Spectrum by choosing a large positive t .

#  Invariant Measure

The invariant measure could be understood as a (probability) distribution of particles in the dynamical system when time tends to infinity. Two methods are used to cross-validate the calculation of the invariant measure.

##  Ulam's Method

The Ulam's method uses Markov Chain.

 Suppose  we have a discrete finite state process, then the invariant measure could be calculated by computing the eigenvalues and eigenvectors of the corresponding Markov matrix. The normalised eigenvector corresponding to eigenvalue 1, is the required invariant measure.

If  we have $X:[a,b]\rightarrow[a,b]$, we could partition the interval $[a,b]$ into N equally spaced subintervals. Then, considering each subinterval as a single state, we have a N-state Markov process.

And the Markov matrix of this process is estimated by:

$p_{ij}=\frac{\text{number of times X(t) in state i and X(t+1) in state j}}{\text{number of times X(t) in state i}}$



However the process we are dealing with is 

$\Phi:C[I,\mathbb{R}]\rightarrow C[I,\mathbb{R}]\quad \text{where } I=[-\tau,0]$

What should we do?

#### Step1:

We have to map $C[I,\mathbb{R}]$ onto $\mathbb{R}$, so we can define a Markov process that we can work with. A natural choice is an equivalence relation on $C[I,\mathbb{R}]$:

$Y_\tau\equiv X_\tau \text{ iff }Y_\tau(0)=X_\tau(0)$

Thus $\mu(X\tau)=X_\tau(0)$ is the required mapping which maps $C[I,\mathbb{R}]$ onto $\mathbb{R}$

#### Step2:

We have to restrict $\mathbb{R}$ to a finite interval $[a,b]$. This is totally for the purpose of numerical simulation. We choose:

b = the largest  value a path has ever visited

a = the smallest value a path has ever visited


Now, we have reduced our problem to what we have discussed at the very beginning of this section. So then we estimate the the Markov matrix and compute its normalised eigenvector with  eigenvalue 1.



##  Birkhoff's Theorem

The Birkhoff's Theorem states that given $(X,\mathscr{F},\mu)$ is a probability space and $f:X\rightarrow X$ a measurable function such that $\mu$ is invariant with respect to f then we have:

$lim(n\rightarrow\infty)\frac{1}{n}\Sigma_{i=0}^{n-1}I_A(f^i(x))=\mu(A)$

This means that if $X=\{x_1,x_2,\dots,x_n\}$ is finite, then

$\mu(X=x_i)=\frac{\text{number of times the path visited }x_i}{\text{number of all discrete time points}}$

We use the same mapping to reduce our infinite dimensional object to one dimension with finite states as what we have done in Ulam's method.

# Another Two Equations

Apart from the Logistic equation, the same scripts were also written for other two equations, which are the Predator-Pray equation, and Wreight's conjecture.

Predator-Pray:

$dx=x(r_1-a_{11}x_\tau-a_{12}y)dt+\sigma xdW$

$dy=y(-r_2+a_{21}x-a_{22}y_\tau)dt+\sigma ydW$


Wreight's conjecture:

$dX=-\alpha X_\tau(1+X)dt+\sigma XdW$


That is all.







 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3OTE1Njg2OTAsLTIyMDMyMTU1MywxMz
k5MzMzMDA4LC04NDA1NTY5NTksMTY0NjQ0MDE0MywxNDEzODQ2
MTAsLTE1Mzc2MjcyOSw4NDg5NTQwNDgsMTE3MzQ1MDc2MCwxNT
c1ODAzMjcyLDg2MjUyNTExOCwtMjc0OTc4MDY2LDE5MTE2MzA5
NTgsLTEwMTM4Mzc5NTAsLTYwODgzNTM0MiwtODY3OTUxNjUsMT
M0MjY3MTg2NCwyNjU4NzQxNDAsMTQ0NjIwMzQ1MSwtNjIxNzAy
MDM1XX0=
-->
