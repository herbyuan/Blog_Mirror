---
title: 主成分分析(PCA)
date: 2023-03-13 18:49:17
tags: PCA
updated:
type:
comments:
description:
keywords:
top_img:
mathjax:
katex:
aside:
aplayer:
highlight_shrink:
---


主成分分析(PCA)

PCA的主要目标是将特征维度变小，同时尽量减少信息损失。PCA通过线性变换将n维原始特征映射到维（k < n）上，称这k维特征为主成分。新构造出的k维特征相互正交，且k维数据尽可能多地包含原始数据的信息。

新的维度要能区分数据，就要求此维度上数据间的方差足够大。构造新的正交向量，使得任何数据投影的第一大方差在第一个坐标(称为第一主成分)上，第二大方差在第二个坐标(第二主成分)上，依次类推。

假设原先的样本数据是m条特征数为n的数据集，排列成n行m列的矩阵 $X = (x_1,x_2,\cdots,x_m)$, 其中 $x_i$ 是表示第 $i$ 条数据的列向量. 

### (维度的)协方差矩阵

一个维度的方差是每个元素与该维度均值的差的平方和的均值，即

$$Var(a) = \frac 1 m \Sigma_{i=1}^m (a_i - \bar a)^2 = \frac 1 m (\Sigma_{i=1}^m a_i^2 - m\bar a^2)$$

如果把每个维度都零均值化，方差就是该行的内积。

同样的，不同维度间的协方差在零均值化后也可以写作两个行向量的内积，于是 $n$ 个维度的协方差矩阵可以写成:

$$C = \frac 1 m X\cdot X^T$$ 

> 注意：如果要无偏估计协方差，分母应当是 $m-1$, 但是此处所有的分数最终会被约掉，所以对计算不带来影响。

**协方差矩阵一定是半正定的**，任取一个向量 $x$, $x^TCx = x^TX \cdot X^Tx = (x^TX)(x^TX)^T \ge 0$.

**实对称矩阵一定可以对角化**，即存在一个正交矩阵 $P$, 使得 $P^TCP = \Lambda$ 是一个对角矩阵，对角线元素就是矩阵的所有特征值。由于 $C$ 半正定，所有的特征值都是非负数，这里不妨设 $\lambda_1 \ge \lambda_2 \ge \cdots \ge \lambda_n \ge 0$.

### 坐标变换

回顾一下坐标变换的知识。样本数据在基 $e_1,e_2,\cdots,e_n$ 下的坐标是 $x_1,x_2,\cdots,x_m$。

另外取一组单位正交基 $\{\eta_1,\eta_2,\cdots,\eta_n\}$, 在此基下原先数据的坐标变换为 $(\eta_1,\eta_2,\cdots,\eta_n)^{-1}(x_1,x_2,\cdots,x_m)$

考虑上述的正交矩阵 $P$, 其逆矩阵为 $P^T$。以 $P$ 为基，此时任何一个原先坐标为 $x$ 的点被对应到 $P^{-1}x = P^Tx$, 整个样本矩阵的坐标在新的基下变成 $P^TX$, 再计算新的基的协方差矩阵，可以看到

$$C_{new} = \frac 1 m (P^TX)(P^TX)^T = \Lambda$$

如果只考虑 $P^T$ 的前 $k$ 行的矩阵 $P_k^T$, 则样本点被从 $n$ 维空间映射到了 $k$ 维，只保留了基中的前 $k$ 维度。此时再计算协方差矩阵，得到 

$$\frac 1 m (P_k^TX)(P_k^TX)^T = \text{diag} \{\lambda_1,\cdots,\lambda_k\}$$

### 一个极端情况

考虑一个极端情况，如果 $\lambda_n = 0$, 在将 $n$ 维坐标变换到 $n-1$ 维时会发生什么。

上述 $P^TCP = \Lambda$, 即 $C(p_1,\cdots, p_n) = P \Lambda = (\lambda_1 p_1, \cdots, \lambda_n p_n)$, 其中 $\{p_1,\cdots, p_n\}$ 构成 $n$ 维线性空间的一组基.

对于任意 $n$ 维向量 $x$, 可以表示成 $x = \Sigma_{i=1}^n x_i p_i$, 所以

$$Cx = C\Sigma_{i=1}^n x_i p_i = \Sigma_{i=1}^n x_i \lambda_i p_i$$

注意到对应的 $\lambda_n = 0$ 时，相对应的项 $x_n$ 并不会影响计算的结果, 因此在计算时直接忽略最后一项也不会有误差。相应地当对应的特征值比较小时，对应的坐标对计算结果的影响也比较小。

同样地，特征值是零的特征向量在坐标变换后对应维度的方差是零，在零均值的前提下意味着所有坐标在这个维度都是0，因此取这一维度完全无法区分各个数据点，和不取的效果是一样的。对于特征值较小的情况同理，各个坐标在这一维度密集分布于零附近，区分度很小。


### PCA 的思路

考虑以 $P$ 为基时，不同的基是正交的，不同维度之间的相关系数也成了零。因此选用这组基来构建新的坐标是合理的。唯一的问题就是维度减小时应当保留哪些维度。

如果只能保留一个维度，为了获得尽可能大的方差，应该选择最大特征值对应的特征向量作为坐标(称为第一主成分)。如果解释力不够，应当增加第二大的特征值对应的特征向量作为坐标……以此类推，直到解释力达到要求。

对于特征值是零的那些维度，上文已经讨论，他们的存在与否完全没有影响，因此可以直接省略。

主成分所占整个信息的百分比可以如下计算：

$$\sqrt{\frac{\Sigma_{i=1}^k \sigma_i^2}{\Sigma_{i=1}^n\sigma_i^2}}$$


### PCA 算法的步骤

假设有m条n维数据。

- 将原始数据排列成n行m列的矩阵，并将矩阵的每一行 (每一个维度) 零均值化后记为 $X$ 

- 求出 $X$ 的协方差矩阵 $C = \frac 1 m XX^t$

- 求出协方差矩阵的特征值以及对应的特征向量

- 将特征向量按照特征值由大到小排列(协方差矩阵是半正定的，对应的所有特征值都大于等于零)，取前 $k$ 行组成矩阵 $P$

- $Y = PX$ 即为降维到 $k$ 之后的数据 


### 简单数值的例子

$$X = \begin{bmatrix}
   -1 & -1 & 0 & 2 & 0 \\
   -2 & 0 & 0 & 1 & 1
\end{bmatrix}$$

现在用PCA方法将二维数据降到一行。

<img src="scatter.png" width="30%">

- 因为 $X$ 矩阵的每行已经是零均值，所以不需要去平均值。

- 求协方差矩阵

$$C = \frac 1 5 \left( \begin{matrix}
   -1 & -1 & 0 & 2 & 0 \\
   -2 & 0 & 0 & 1 & 1
\end{matrix}\right) 
\left( \begin{matrix}
   -1 & -2 \\
   -1 & 0 \\
   0 & 0 \\
   2 & 1 \\
   0 & 1
\end{matrix}\right)
 = \left( \begin{matrix}
   \frac 6 5 & \frac 4 5 \\
   \frac 4 5 & \frac 6 5
\end{matrix}\right) $$

- 求协方差矩阵的特征值与特征向量, 得到特征向量为 $\lambda_1 = 2, \lambda_2 = \frac 2 5$, 对应的正交单位特征向量为

$$p_1 = \left( \begin{matrix}
   \frac {\sqrt 2} 2\\
   -\frac {\sqrt 2} 2
\end{matrix}\right),
p_2 = \left( \begin{matrix}
   \frac {\sqrt 2} 2\\
   \frac {\sqrt 2} 2
\end{matrix}\right)$$

-矩阵 $P$ 为

$$P = \left( \begin{matrix}
   \frac {\sqrt 2} 2 & \frac {\sqrt 2} 2\\
   -\frac {\sqrt 2} 2 & \frac {\sqrt 2} 2
\end{matrix}\right)$$

- 用 $P$ 的第一行乘以矩阵 $X$, 得到降维后的表示为

$$Y= \left( \begin{matrix}
   \frac {\sqrt 2} 2 & \frac {\sqrt 2} 2
\end{matrix}\right)
\left( \begin{matrix}
    -1 & -1 & 0 & 2 & 0 \\
    -2 & 0 & 0 & 1 & 1
\end{matrix}\right)
= \left( \begin{matrix}
    -\frac{3 \sqrt 2}{2} & -\frac{\sqrt 2}{2} & 0 & \frac{3 \sqrt 2}{2} & -\frac{\sqrt 2}{2} 
\end{matrix}\right)
$$

可见，新的坐标轴是过原点的45度的斜线，所有点的投影在这条线上最分散。

引用一个网上的python程序代替人肉计算：

```python

# Python实现PCA
import numpy as np
def pca(X,k):# k is the components you want
  # mean of each feature
  n_samples, n_features = X.shape
  mean=np.array([np.mean(X[:,i]) for i in range(n_features)])
  # normalization
  norm_X=X-mean
  # scatter matrix
  scatter_matrix=np.dot(np.transpose(norm_X),norm_X)
  # Calculate the eigenvectors and eigenvalues
  eig_val, eig_vec = np.linalg.eig(scatter_matrix)
  eig_pairs = [(np.abs(eig_val[i]), eig_vec[:,i]) for i in range(n_features)]
  # sort eig_vec based on eig_val from highest to lowest
  eig_pairs.sort(reverse=True)
  # select the top k eig_vec
  feature=np.array([ele[1] for ele in eig_pairs[:k]])
  # get new data
  data=np.dot(norm_X,np.transpose(feature))
  return data

X = np.array([[-1, -2], [-1, 0], [0, 0], [2, 1], [0, 1]])

print(pca(X,1))

```



### PCA 的一些性质

**缓解维度灾难：**PCA 算法通过舍去一部分信息之后能使得样本的采样密度增大（因为维数降低了），这是缓解维度灾难的重要手段；

**降噪：**当数据受到噪声影响时，最小特征值对应的特征向量往往与噪声有关，将它们舍弃能在一定程度上起到降噪的效果；

**过拟合：**PCA 保留了主要信息，但这个主要信息只是针对训练集的，而且这个主要信息未必是重要信息。有可能舍弃了一些看似无用的信息，但是这些看似无用的信息恰好是重要信息，只是在训练集上没有很大的表现，所以 PCA 也可能加剧了过拟合；

**特征独立：**PCA 不仅将数据压缩到低维，它也使得降维之后的数据各特征相互独立；


### PCA 与 SVD

特征值和特征向量是针对方阵才有的，而对任意形状的矩阵都可以做奇异值分解。

SVD：矩阵的奇异值分解其实就是对于矩阵 $A$ 的协方差矩阵 $A^TA$ 和 $AA^T$ 做特征值分解推导出来的：

$$A_{m,n} = U_{m,m}\Lambda_{m,n}V_{n,n}^T \approx U_{m,k}\Lambda_{k,k}V_{k,n}^T$$

其中：$U, V$ 都是正交矩阵。这里的约等于是因为 $\Lambda$ 中有 $n$ 个奇异值，但是由于排在后面的很多接近 $0$，所以我们可以仅保留比较大的 $k$ 个奇异值。

$$A^TA = (U\Lambda V^T)^T(U\Lambda V^T) = V\Lambda^T U^T U \Lambda V^T = V\lambda^2V^T$$

$$AA^T = (U\Lambda V^T)(U\Lambda V^T)^T = U\Lambda V^TV\Lambda^TU^T = U\Lambda^2U^T$$

所以，$V,U$ 两个矩阵分别是 $A^TA, AA^T$ 的特征向量，中间的矩阵对角线的元素是 $A^TA$ 和 $AA^T$ 的特征值。我们也很容易看出 $A$ 的奇异值和 $A^TA$ 的特征值之间的关系。

PCA 需要对协方差矩阵 $C = \frac 1 m XX^T$ 进行特征值分解； SVD 也是对 $A^TA$ 进行特征值分解。如果取 $A = \frac{X^T}{\sqrt m}$ 则两者基本等价。所以 PCA 问题可以转换成 SVD 求解。

而实际上 Sklearn 的 PCA 就是用 SVD 进行求解的，原因有以下几点：

- 当样本维度很高时，协方差矩阵计算太慢；
- 方阵特征值分解计算效率不高；
- SVD 除了特征值分解这种求解方式外，还有更高效更准确的迭代求解方式，避免了 $A^TA$ 的计算；
- 其实 PCA 与 SVD 的右奇异向量的压缩效果相同。


```python
##用sklearn的PCA
from sklearn.decomposition import PCA
import numpy as np
X = np.array([[-1, 1], [-2, -1], [-3, -2], [1, 1], [2, 1], [3, 2]])
pca=PCA(n_components=1)
pca.fit(X)
print(pca.transform(X))

```

sklearn 中的 PCA 是通过 `svd_flip` 函数实现的，sklearn对奇异值分解结果进行了一个处理，因为$u_i*σ_i*v_i=(-u_i)*σ_i*(-v_i)$，也就是 $u$ 和 $v$ 同时取反得到的结果是一样的，而这会导致通过 PCA 降维得到不一样的结果（虽然都是正确的）。
