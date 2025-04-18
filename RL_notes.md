## 强化学习入门

### 1 基本原理

王树森老师[视频链接](https://www.bilibili.com/video/BV12o4y197US/?spm_id_from=333.788.videopod.episodes&vd_source=2173cb93b451f2278a1c87becf3ef529)

#### 1.1 基本概念

##### 1.1.1 概率论基础知识

- 随机变量(Random Variable)，记作 $X$

- 观察值，记作 $x$

- 概率密度函数 $p(x)$

- 期望 $$E(X)=\int_{-\infty}^{+\infty}xf(x)dx$$

- 随机抽样

##### 1.1.2 Terminology

- $\text{state}$ $s$ : 状态，由环境所决定
- $\text{action}$ $a$ : 动作
- $\text{agent}$ : 智能体
- $\text{policy}\ \pi$ : 决策， $$\pi(a|s) = P(A = a| S = s)$$

- $\text{reward}$ $R$ : 奖励，通常需要自己定义
- $\text{state transition}$ :  $$\text{old state} \xrightarrow{\text{action}}\text{new state}$$ 具有随机性

##### 1.1.3 Randomness in Reinforcement Learning

- $\text{Actions have randomness}$ : 动作是根据 $\text{policy}$ $\pi$ 随机抽样得到的
- $\text{State transitions have randomness}$ : 下一个 $S'$ 是由状态转移函数筹集抽样的 

##### 1.1.4 Rewards and Returns

- $\text{Return}$ : $\text{cumulative future reward}$ 

  - $$U_t=R_t+R_{t+1}+R_{t+2}+\cdots$$

- $\text{Discounted return}$ 

  - $\gamma$ : 折扣率考虑未来奖励影响的承担，属于超参数

  - $$U_t=\sum_{k = t}^{\infty}\gamma^{k - t}R_k$$

##### 1.1.5 Randomness in Returns

- 动作随机 : $$\mathbb{P}[\textcolor{red}{A=a}\ \big|\ \textcolor{green}{S=s}]=\pi(\textcolor{red}a\ |\ \textcolor{green}s)$$
- 状态随机 : $$\mathbb{P}\left[\textcolor{green}{S'=s'} \big| \textcolor{red}{A=a}, \textcolor{green}{S=s}\right] = p(\textcolor{green}{s'} | \textcolor{green}{s},\textcolor{red}{a})$$  

##### 1.1.6 Action-Value Function $$Q(\textcolor{green}{s},\textcolor{red}{a})$$

- $$Q_\pi(\textcolor{green}{s_t},\textcolor{red}{a_t})=\mathbb{E}[U_t|\textcolor{green}{S_t=s_t},\textcolor{red}{A_t=a_t}]$$
- 由于 $U_t$ 具有随机性，由 $\color{red}A_t,A_{t+1},A_{t+2},\cdots$ 和 $\color{green}S_t,S_{t+1},S_{t+2},\cdots$ 所决定
- $\mathbb{E}$ : 求期望可以将 $t$ 以后的随机性积分掉，变为一个确切的值
- 直观意义：如果用 $\text{policy}\ \pi$ 在 $\textcolor{green}{s_t}$ 状态下，做 $\textcolor{red}{a_t}$ 是好还是坏，根据分数

##### 1.1.7 Optimal action-value function $Q^*(\textcolor{green}{s},\textcolor{red}{a})$

- 
