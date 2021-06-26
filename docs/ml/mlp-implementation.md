> 本文是对[thomasjungblut-common](https://github.com/thomasjungblut/thomasjungblut-common)机器学习库代码的学习笔记
### 概念

**Optimizer**

也叫`Minimizer`，通过多轮迭代，在初始化参数`theta`之上利用代价函数`f`优化得到最优参数

```java
public DoubleVector minimize(CostFunction f, DoubleVector theta,int maxIterations);
```

> `f`: 代价函数
>
> `theta`: 初始化参数
>
> `maxIterations`：最大迭代次数

**CostFunction**

代价函数，也叫`LossFunction`，实现有`CrossEntropyLoss`，`HingeLoss`，`LogLoss`等。`LossFunction`需要实现`loss`计算和`gradient`计算

```java
public double calculateLoss(DoubleMatrix y, DoubleMatrix hypothesis);

public double calculateLoss(DoubleVector y, DoubleVector hypothesis);
```

> `y`：样本真实label
>
> `hypothesis`：根据当前`weights`预测的样本label

### 初始化训练

#### 初始化

**定义CostFunction**

```java
CostFunction costFunction = new MultilayerPerceptronCostFunction(this, features, outcome);
```

**定义MLP**

在定义MLP时，通过如下公式初始化`weights`

```mathematica
weights by SQRT(6)/((num units * left layer) + (num units right layer)) 
```

初始化`layer[0]`和`layer[1]`的`weights`参数

- `layer[0] weights`

  > layer[0] -> layer[1]
  >
  > size = 4 * (3 + 1)

  ```properties
  [0.48648333467201565, 0.6925038518707476, -0.6407451573377947]
  [0.18363294640677275, -0.6318339104975048, -0.3377795511428756]
  [-0.2843174571626119, 0.09247849165866828, 0.8489541800856073]
  [0.0547363277507964, 0.06548406260296069, 0.7310565004799185]
  ```

- `layer[1] weights`

  > layer[1] -> layer[2]
  >
  > size = 1 * (4 + 1)

  ```properties
  [0.532915792500396, 0.20115981409625333, -0.3114541695609426, 0.05996064284821445, 0.7585999616576491]
  ```

**执行训练**

```java
trainInternal(minimizer, maxIterations, verbose, costFunction, initialTheta);
```

### 训练

**执行`minimize`优化**

使用优化器`GradientDescent`执行最小化cost来进行梯度计算。

**初始化Cost**

初始化`lastCosts`为`Double.MAX_VALUE`，`lastCosts`保留最近三次的代价值

**循环迭代更新参数theta**

- 计算当前Iteration样本的`Cost`

  调用`AbstractMiniBatchCostFunction`中`evaluateCost`计算cost，如果输入的样本大于1，就启用多线程进行Cost计算；

  如果样本大小等于1，那么直接通过`MultilayerPerceptronCostFunction`的`evaluateBatch`方法评估cost。

- **`MultilayerPerceptronCostFunction`**

  `CostFunction`中主要的成员

  ```java
  @param input the input parameters (theta).
  @param x     the features.
  @param y     the outcome.
  ```

### 计算梯度

**反向传播计算梯度**

执行`backwardPropagate`计算梯度delta

- 输出层delta为最后一层预测值和实际样本label的差值

  ```java
  deltaX[deltaX.length - 1] = ax[conf.layerSizes.length - 1].subtract(y);
  ```

- 其他处输入层以外，通过激活函数计算delta

  ```java
  // apply the gradient of the activations
  deltaX[i] = deltaX[i].multiplyElementWise(conf.activations[i].gradient(zx[i]));
  ```

- 计算梯度deltaX

  ```properties
  deltaX[1]: [0.03541665788543774, -0.0506673018850211, 0.010198874057206379, 0.12032269108680421]
  deltaX[2]: [0.756498544961436]
  ```

**更新梯度**

执行`calculateGradients`更新梯度

- deltaX[i + 1]的转置与ax[i]相乘得到每个layer的梯度

  ```java
  DoubleMatrix gradDXA = multiply(deltaX[i + 1], ax[i], true, false, conf);
  ```

调用对应`Loss Function`计算损失

#### 继续Iteration

当前Iteration样本的`Cost`计算完成后

- 在`lastCosts`中更新最新一次计算的`Cost`

- 更新上一次的参数`lastTheta`

  ```java
  theta = theta.subtract(gradient.multiply(alpha));
  ```

  如果优化器有`momentum`，那么`weight`更新需要增加动量

  ```java
  theta = theta.add((lastTheta.subtract(theta)).multiply(momentum));
  ```

### 样例数据

**MLP定义**

```properties
optimizer: SGD
learning rate: 0.05
momentum: 0.01
mlp layers: [2, 4, 1]
activation function: [sigmoid, sigmoid, logloss]
```

**样本**

```properties
features: [1.0, 1.0]
lables: [0.0]
```

#### 第一个iteration结束后

**Cost**

```properties
cost: 1.4126323609596292
```

**Gradient**

```properties
- layer[0]: [[0.03541665788543774, -0.0506673018850211, 0.010198874057206379]
			[0.12032269108680421, 0.03541665788543774, -0.0506673018850211]
            [0.010198874057206379, 0.12032269108680421, 0.03541665788543774]
            [-0.0506673018850211, 0.010198874057206379, 0.12032269108680421]]
- layer[1]: [0.756498544961436, 0.47765573390039656, 0.23680854125646125, 0.49823906811878865, 0.5301806054454972]
```

**Weights**

```properties
- weights[0]: [[0.4847302101066865, 0.1861409778500813, -0.28482230142844367],
				[0.04878035454199959, 0.6907507273054184, -0.6293258790541962],
				[0.09197364739283656, 0.05952808939416388, -0.6424982819031239],
                [-0.33527151969956703, 0.8484493358197757, 0.7251005272711217]]
- weights[1]: [0.4954691145248049, 0.1775158552681837, -0.32317619235313744, 0.03529780897633441, 0.732356021688097]
```



#### 第二个iteration结束后

**Cost**

```properties
cost: 1.342209938349636
```

**Gradient**

```properties
- layer[0]: [0.03056187590540195, 0.03056187590540195, 0.03056187590540195]
			[-0.05148356571261431, -0.05148356571261431, -0.05148356571261431]
			[0.005865717620609249, 0.005865717620609249, 0.005865717620609249]
			[0.11424147298985331, 0.11424147298985331, 0.11424147298985331]

- layer[1]: [0.7387323552705708, 0.4655332474975401, 0.23244409914241856, 0.486286419661338, 0.5149520456646565]
```

**Weights**

```properties
- weights[0]: [[0.48321739724936913, 0.18868941435285572, -0.2851126544506638],
				[0.04312540162900185, 0.689237914448101, -0.6267774425514219], 
				[0.0916832943706164, 0.053873136481166144, -0.6440110947604413], 
				[-0.3327230831967926, 0.8481589827975554, 0.719445574358124]]
- weights[1]: [0.45890186293891166, 0.15447195951705545, -0.3346821752606871, 0.011226631203098176, 0.7068658954276965]
```

