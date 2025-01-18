---
mathjax: true
cover: 'https://9258ce6.webp.li/post1/lstm.png'
banner: 'https://9258ce6.webp.li/post1/lstm.png'
title: LSTM反向传播推导及C++代码实现
date: LSTM
tags: LSTM反向传播
categories: 深度学习
abbrlink: 1552645455
---
之前在找资料时候发现大多数文章都只讲了LSTM前向传播，比较少有关于反向传播的内容，而且反向传播有很多公式，需要一步步地推才容易理解，另外在实现过程中还有一些小细节需要注意。

<!-- more -->

## 1 LSTM基础概念

**时间步（time_steps)：**可以认为是输入的次数，有多少次输入就有“多少个”LSTM单元，但从本质上来看却是一个LSTM单元的复用。

如同简单RNN一样：

<!-- ![rnn](/pic/post1/rnn.png) -->

{% image https://9258ce6.webp.li/post1/rnn.png fancybox:true %}

只有一个单元，但为了分析起来简单，所以通常将它展开，展开后看似有多个单元，但其实这些单元内部的权重完全一样，只不过中间状态和输入不一样。

**输入维度(input_size)：**每个单元接收的输入shape是1×input_size。

**隐藏层大小/输出维度（hidden_size/output_size)：**隐藏层大小决定了ht和Ct的大小，由于输出和ht相等，所以也决定了输出的大小。

**例：**假如存在一个任务，需要通过前面10个时刻的值来预测第11个时刻的值，此时time_steps为10，input_size为1（每次输入1个值，输入10次）。

## 2 LSTM权重大小计算

前面提到，本质上只有一个LSTM单元，因此权重大小和time_steps无关，只与input_size和hidden_size有关。

LSTM单元需要计算三个门+一个候补细胞状态，每次都是分别将输入(1×input_size)和ht(1×hidden_size)与对应的权重进行矩阵相乘，输出大小都为1×hidden_size，最后加上偏置，因此可以推得权重大小为：

$$ P=4\times(inputsize\times hiddensize+hiddensize\times hiddensize+hiddensize) $$

## 3 LSTM反向传播推导

### 3.1 最后一个LSTM单元

LSTM最后一个单元其ht的梯度只来自输出。

<!-- ![lstm1](/pic/post1/lstm1.png) -->

{% image https://9258ce6.webp.li/post1/lstm1.png fancybox:true %}

#### 3.1.1 输出门ot的反向传播

由于输出门计算公式为：

$$h_t=o_t\cdot\tanh(C_t) \tag{1}$$

因此求输出门ot的梯度时，只需要将ht的梯度与tanh(Ct)相乘（点乘），这里tanh(Ct)的结果在前向传播时计算得到，因此需要在前向传播时将其保存下来：

$$\frac{\partial L}{\partial o_t}=\frac{\partial L}{\partial h_t}\cdot\tanh(C_t)$$

接着求输出门ot内部的梯度，由于ot门计算公式为：

$$o_t=\sigma(x_t\times W_{ox}+h_{t-1}\times W_{oh}+b_o)$$

所以求ot门内部的梯度，即可得到bo的梯度，由于sigmoid的导数可以表示为：

$$S^{'}(x)=S(x)(1-S(x))$$

所以，bo的梯度为：

$$\frac{\partial L}{\partial b_o}=\frac{\partial L}{\partial h_t}\cdot\tanh(C_t)\cdot(o_t\cdot(1-o_t))$$

这里先提一下矩阵乘法的梯度计算公式：

$$Y=X\times W$$

$$\frac{\partial L}{\partial X}=\frac{\partial L}{\partial Y}\times W^T$$

$$\frac{\partial L}{\partial W}=X^T\times\frac{\partial L}{\partial Y}$$

由此可以进一步计算出$x_t、W_{ox}、h_{t-1}、W_{oh}$的梯度分别为：

$$\frac{\partial L}{\partial h_{t-1}}=\left[\frac{\partial L}{\partial h_t}\cdot\tanh(C_t)\cdot(o_t\cdot(1-o_t))\right]\times W_{oh}^T$$

$$\frac{\partial L}{\partial x_t}=\left[\frac{\partial L}{\partial h_t}\cdot\tanh(C_t)\cdot(o_t\cdot(1-o_t))\right]\times W_{ox}^T$$

$$\frac{\partial L}{\partial W_{oh}}=h_{t-1}^T\times\left[\frac{\partial L}{\partial h_t}\cdot\tanh(C_t)\cdot\left(o_t\cdot(1-o_t)\right)\right]$$

$$\frac{\partial L}{\partial W_{ox}}=x_t^T\times\left[\frac{\partial L}{\partial h_t}\cdot\tanh(C_t)\cdot\left(o_t\cdot(1-o_t)\right)\right]$$

#### 3.1.2 其它门的反向传播

由公式（1）可知，tanh(Ct)的梯度为ht的梯度与输出门ot的点积，根据tanh函数的梯度（$(\tanh x)'=1-\tanh^2x$）可以得到Ct的梯度为：

$$\frac{\partial L}{\partial C_t}=\frac{\partial L}{\partial h_t}\cdot o_t\cdot(1-tanh^2(C_t)) \tag{2}$$

<!-- ![lstm2](/pic/post1/lstm2.png) -->

{% image https://9258ce6.webp.li/post1/lstm2.png fancybox:true %}

由于Ct计算公式为：

$$C_t=f_t\cdot C_{t-1}+i_t\cdot \stackrel{\sim }{C_t}$$

所以可以得到$f_t、C_{t-1}、i_t、\stackrel{\sim }{C_t}$的梯度：

$$\frac{\partial L}{\partial f_t}=\frac{\partial L}{\partial C_t}\cdot C_{t-1}$$

$$\frac{\partial L}{\partial C_{t-1}}=\frac{\partial L}{\partial C_t}\cdot f_t \tag{3}$$

$$\frac{\partial L}{\partial i_t}=\frac{\partial L}{\partial C_t}\cdot \stackrel{\sim }{C_t}$$

$$\frac{\partial L}{\partial \stackrel{\sim }{C_t}}=\frac{\partial L}{\partial C_t}\cdot i_t$$

现在得到了输入门、遗忘门、候补细胞状态的梯度后，可以按照和输出门一样的方法，先计算激活函数内部的梯度，再根据矩阵乘法的梯度公式进一步计算出$W、h_{t-1}、x_t$的梯度，这里不再赘述。

由于$h_{t-1}和x_t$在前向传播中被使用了4次，因此这里也会收到来自各个门和候补细胞状态的4个梯度，将其相加即可得到总梯度。

### 3.2 其它LSTM单元的反向传播

其它LSTM单元来说，其与最后一个LSTM单元的区别在于ht和Ct的梯度来源不同。

前面提到，最后一个LSTM单元的ht只来自输出，而在3.1.2的最后，我们计算出了最后一个LSTM单元对$h_{t-1}$的梯度，因此倒数第二个LSTM单元则需要加上该梯度。此外，其输出方向也可能有梯度传来，比如多层LSTM的情况：

<!-- ![multi_LSTM](/pic/post1/multi_LSTM.png) -->

{% image https://9258ce6.webp.li/post1/multi_LSTM.png fancybox:true %}

此时，下层LSTM每一个单元的输出，是上层每一个LSTM单元的输入。因此上层LSTM单元计算完$x_t$的梯度后，将会传回给下层LSTM单元。

同样的，Ct的梯度也来自两个方向，一方面来自当前LSTM单元中ht传来的梯度，如公式(2)；另一方面来自下一个LSTM单元，如公式(3)。

### 3.3 权重的更新

前面说过，每一个LSTM单元都使用相同的权重，按照链式法则，这些权重也要接收来自每一个单元传来的梯度。

前面已经介绍了LSTM单元如何计算各个门和细胞状态的权重($W_h、W_x$以及$b$)的梯度，但是此时并不能更新这些权重，因为在前面的单元中，计算$h_{t-1}$和$x_t$的梯度时，需要用到这些权重，如果在后面的LSTM单元中修改了权重，那么前面的单元就无法得知原先的权重是多少。{% note color:cyan 正确的做法是将每个单元计算出的权重的梯度累计起来，最后再更新权重参数。%}

## 4 LSTM反向传播的C++实现

为了简单易懂，这里只用最基础的数组方式去实现。

### 4.1 矩阵乘法等基本函数实现

矩阵乘法实现，经典的三个for循环，使用template指定输入输出数组的大小从而实现各种大小的矩阵相乘（后面全部函数都采用这种template的方法）：
{% folding child:codeblock open:false color:blue 矩阵乘法实现 %}
{% box color:light %}
```c++
template<unsigned char row1,unsigned char col1,unsigned char col2>
void matrix_multiply(float inputarr1[row1][col1],float inputarr2[col1][col2],float outputarr[row1][col2]) {
	for(int i=0;i<row1;i++){
		for(int j=0;j<col2;j++){
			float temp=0.0;
			for(int k=0;k<col1;k++)
				temp += inputarr1[i][k] * inputarr2[k][j];
			outputarr[i][j] = temp;
		}
	}
}
```
{% endbox %}
{% endfolding %}

$A^T\times B$的实现，可以理解为第一个矩阵的某一列与第二个矩阵的每一列相乘，作为结果的某一行的每一列数据，结果的行个数与输入的列个数一致：

{% folding 图片演示 color:blue %}
<!-- ![AT_B](/pic/post1/AT_B.png) -->
{% image https://9258ce6.webp.li/post1/AT_B.png fancybox:true %}
{% endfolding %}

{% folding child:codeblock open:false color:blue 代码实现 %}
{% box color:light %}
```c++
template<unsigned char row1,unsigned char col1,unsigned char col2>
void matrix_transpose1_multiply(float inputarr1[row1][col1],float inputarr2[row1][col2],float outputarr[col1][col2])
{
	for(int i=0;i<col1;i++){
		for(int j=0;j<col2;j++){
			float temp = 0.0;
			for(int k=0;k<row1;k++)
				temp += inputarr1[k][i] * inputarr2[k][j];
			outputarr[i][j] = temp;
		}
	}
}
```
{% endbox %}
{% endfolding %}

$A\times B^T$的实现，可以理解为第一个矩阵的某一行与第二个矩阵的每一行相乘，作为结果的某一行的每一列数据，结果的行个数与输入的行个数一致：

{% folding 图片演示 color:blue %}
<!-- ![A_BT](/pic/post1/A_BT.png) -->
{% image https://9258ce6.webp.li/post1/A_BT.png fancybox:true %}
{% endfolding %}

{% folding child:codeblock open:false color:blue 代码实现 %}
{% box color:light %}
```c++
template<unsigned char row1,unsigned char col1,unsigned char row2>
void matrix_transpose2_multiply(float inputarr1[row1][col1],float inputarr2[row2][col1],float outputarr[row1][row2])
{
	for(int i=0;i<row1;i++){
		for(int j=0;j<row2;j++){
			float temp = 0.0;
			for(int k=0;k<col1;k++)
				temp += inputarr1[i][k] * inputarr2[j][k];
			outputarr[i][j] = temp;
		}
	}
}
```
{% endbox %}
{% endfolding %}

### 4.2 反向传播的实现代码

#### 4.2.1 计算门和候补细胞状态的梯度

由于计算各个门和候补细胞状态梯度的过程是类似的，因此将其封装为一个函数：

{% folding child:codeblock open:false color:blue 计算门和候补细胞状态的梯度 %}
{% box color:light %}

```c++
template<int input_dims,int output_dims>
void compute_doors_gredient(float delta[1][output_dims],float K[1][output_dims],float gate[1][output_dims],
	float h_weight[output_dims][output_dims],float x_weight[input_dims][output_dims],
	float h_input[1][output_dims],float x_input[1][input_dims],
	float ht_delta[1][output_dims],float input_delta[1][input_dims],
	float delta_wx[input_dims][output_dims],float delta_wh[output_dims][output_dims],float delta_wb[output_dims],
	int choice=0)
{
	float delta_gate[1][output_dims];
	dot_product<1,output_dims>(delta,K,delta_gate);
	
	float delta_inner[1][output_dims];
	if(choice==0)
		sigmoid_delta<1,output_dims>(delta_gate,gate,delta_inner);
	else
		tanh_delta<1,output_dims>(delta_gate,gate,delta_inner);

	float temp_delta_wx[input_dims][output_dims];
	float temp_delta_wh[output_dims][output_dims];
	
	matrix_transpose2_multiply<1,output_dims,output_dims>(delta_inner,h_weight,ht_delta);
	matrix_transpose2_multiply<1,output_dims,input_dims>(delta_inner,x_weight,input_delta);
	matrix_transpose1_multiply<1,input_dims,output_dims>(x_input,delta_inner,temp_delta_wx);
	matrix_transpose1_multiply<1,output_dims,output_dims>(h_input,delta_inner,temp_delta_wh);
	
    for(int i=0;i<output_dims;i++){
		delta_wb[i] += delta_inner[0][i];
	}

	for(int i=0;i<input_dims;i++)
		for(int j=0;j<output_dims;j++)
			delta_wx[i][j] += temp_delta_wx[i][j];
	
	
	for(int i=0;i<output_dims;i++)
		for(int j=0;j<output_dims;j++)
			delta_wh[i][j] += temp_delta_wh[i][j];
}
```
{% endbox %}
{% endfolding %}

在计算输出门时，delta为$\frac{\partial L}{\partial h_t}$，其余均为$\frac{\partial L}{\partial C_t}$。

K对于输出门来说是$tanh(C_t)$，对于输入门来说是$\stackrel{\sim }{C_t}$，依此类推，忘记的可以回头看前面公式，都有一个点积的操作，而后才是求激活函数的梯度，最后求各个权重和输入的梯度。部分函数并未给出，写起来也很简单，比如点积就是矩阵对应位置相乘。

这里权重的梯度使用“+=”也是前面提到的，权重的梯度需要先累加起来。

#### 4.2.2 LSTM反向传播的主函数

{% folding child:codeblock open:false color:blue LSTM反向传播主函数 %}
{% box color:light %}
```c++
template<int timesteps,int input_dims,int output_dims>
void lstm_backpropagation(float learning_rate,float info[timesteps][6*output_dims],float delta[timesteps][output_dims],
	float input[timesteps][input_dims],float cells_output[timesteps][output_dims],float first_ht[1][output_dims],
	float wx_i[input_dims][output_dims],float wh_i[output_dims][output_dims],float b_i[output_dims],
	float wx_f[input_dims][output_dims],float wh_f[output_dims][output_dims],float b_f[output_dims],
	float wx_c[input_dims][output_dims],float wh_c[output_dims][output_dims],float b_c[output_dims],
	float wx_o[input_dims][output_dims],float wh_o[output_dims][output_dims],float b_o[output_dims],
	float delta_input[timesteps][input_dims])
{
	float delta_ct[1][output_dims];
	float delta_ht[1][output_dims];
	
	//save total loss
	float delta_wx_i[input_dims][output_dims];float delta_wh_i[output_dims][output_dims];float delta_b_i[output_dims];
	float delta_wx_f[input_dims][output_dims];float delta_wh_f[output_dims][output_dims];float delta_b_f[output_dims];
	float delta_wx_c[input_dims][output_dims];float delta_wh_c[output_dims][output_dims];float delta_b_c[output_dims];
	float delta_wx_o[input_dims][output_dims];float delta_wh_o[output_dims][output_dims];float delta_b_o[output_dims];
	
	//initial
	for(int i=0;i<input_dims;i++){
		for(int j=0;j<output_dims;j++){
			delta_wx_i[i][j] = 0;	delta_wx_f[i][j] = 0;	delta_wx_c[i][j] = 0;	delta_wx_o[i][j] = 0; 
		}
	}
	
	for(int i=0;i<output_dims;i++){
		for(int j=0;j<output_dims;j++){
			delta_wh_i[i][j] = 0;	delta_wh_f[i][j] = 0;	delta_wh_c[i][j] = 0;	delta_wh_o[i][j] = 0;
		}
		delta_b_i[i] = 0;	delta_b_f[i] = 0;	delta_b_c[i] = 0;	delta_b_o[i] = 0;
		delta_ht[0][i] = 0;	delta_ct[0][i] = 0;
	}
	
	for(int i=timesteps-1;i>=0;i--)
	{
		float old_ct[1][output_dims];
		float input_gate[1][output_dims];
		float forget_gate[1][output_dims];
		float prepared_cell[1][output_dims];
		float output_gate[1][output_dims];
		float tanc_t[1][output_dims];
		float input_x[1][input_dims];
		float input_h[1][output_dims];
		//recover information
		for(int j=0;j<output_dims;j++){
			old_ct[0][j]        = info[i][j];
			input_gate[0][j]    = info[i][output_dims+j];
			forget_gate[0][j]   = info[i][2*output_dims+j];
			prepared_cell[0][j] = info[i][3*output_dims+j];
			output_gate[0][j]   = info[i][4*output_dims+j];
			tanc_t[0][j]        = info[i][5*output_dims+j];
			if(i!=0)
				input_h[0][j]   = cells_output[i-1][j];   // old ht
			else
				input_h[0][j] = first_ht[0][j];
		}
		
		for(int j=0;j<input_dims;j++)
			input_x[0][j] = input[i][j];
		
		//the final cell's del_ht only from delta
		//other's delta_ht from latter del_ht and delta
		for(int j=0;j<output_dims;j++){
			delta_ht[0][j] += delta[i][j];
		}
		
		//the final cell's del_ct only from ht
		//other's delta_ct from latter_cell's del_ct and current del_ht
		float temp_delta_ct[1][output_dims];
		dot_product<1,output_dims>(delta_ht,output_gate,temp_delta_ct);
		tanh_delta<1,output_dims>(temp_delta_ct,tanc_t,temp_delta_ct);
		for(int j=0;j<output_dims;j++){
			delta_ct[0][j] += temp_delta_ct[0][j];
		}
		
		float delta_ht_i[1][output_dims];
		float delta_ht_f[1][output_dims];
		float delta_ht_c[1][output_dims];
		float delta_ht_o[1][output_dims];
		float delta_input_i[1][input_dims];
		float delta_input_f[1][input_dims];
		float delta_input_c[1][input_dims];
		float delta_input_o[1][input_dims];
		compute_doors_gredient<input_dims,output_dims>(delta_ct,prepared_cell,input_gate,   wh_i,wx_i,input_h,input_x,delta_ht_i,delta_input_i,delta_wx_i,delta_wh_i,delta_b_i);
		compute_doors_gredient<input_dims,output_dims>(delta_ct,old_ct,       forget_gate,  wh_f,wx_f,input_h,input_x,delta_ht_f,delta_input_f,delta_wx_f,delta_wh_f,delta_b_f);
		compute_doors_gredient<input_dims,output_dims>(delta_ct,input_gate,   prepared_cell,wh_c,wx_c,input_h,input_x,delta_ht_c,delta_input_c,delta_wx_c,delta_wh_c,delta_b_c,1);
		compute_doors_gredient<input_dims,output_dims>(delta_ht,tanc_t,       output_gate,  wh_o,wx_o,input_h,input_x,delta_ht_o,delta_input_o,delta_wx_o,delta_wh_o,delta_b_o);
		
		//compute delta_ht for previous cell
		for(int j=0;j<output_dims;j++)
			delta_ht[0][j] = delta_ht_i[0][j]+delta_ht_f[0][j]+delta_ht_c[0][j]+delta_ht_o[0][j];
		
		//compute delta_input(if use multi-layer lstm)
		for(int j=0;j<input_dims;j++)
			delta_input[i][j] = delta_input_i[0][j]+delta_input_f[0][j]+delta_input_c[0][j]+delta_input_o[0][j];
		
		//compute delta_ct for previous cell
		dot_product<1,output_dims>(delta_ct,forget_gate,delta_ct);
	}
	
	//update weight
	for(int i=0;i<input_dims;i++){
		for(int j=0;j<output_dims;j++){
			wx_i[i][j] -= learning_rate*delta_wx_i[i][j];
			wx_f[i][j] -= learning_rate*delta_wx_f[i][j];
			wx_c[i][j] -= learning_rate*delta_wx_c[i][j];
			wx_o[i][j] -= learning_rate*delta_wx_o[i][j];
		}
	}
	
	for(int i=0;i<output_dims;i++){
		for(int j=0;j<output_dims;j++){
			wh_i[i][j] -= learning_rate*delta_wh_i[i][j];
			wh_f[i][j] -= learning_rate*delta_wh_f[i][j];
			wh_c[i][j] -= learning_rate*delta_wh_c[i][j];
			wh_o[i][j] -= learning_rate*delta_wh_o[i][j];
		}
		b_i[i] -= learning_rate*delta_b_i[i];
		b_f[i] -= learning_rate*delta_b_f[i];
		b_c[i] -= learning_rate*delta_b_c[i];
		b_o[i] -= learning_rate*delta_b_o[i];
	}
}	
```
{% endbox %}
{% endfolding %}

“for(int i=timesteps-1;i>=0;i- -)”即是从最后一个LSTM单元开始计算梯度：

首先需要还原前向传播计算的信息，比如前面说的计算输出门ot的梯度时，需要知道$tanh(C_t)$的值，计算ot门内部梯度时，也需要知道ot的值，其余同理。对于$h_{t-1}$，我在前向传播时将每一个LSTM单元的输出保存在cells_output中，因此代码中“i-1”代表上一次LSTM单元的输出。而对于第一个LSTM单元，如果ht和Ct采用零初始化，那么这里$h_{t-1}$就为0，但是Stateful LSTM等情况时，会出现不为0的情况，因此这里直接增加了一个参数，将最开始的ht传进来。

参数delta是每一个LSTM单元的输出方向传来的梯度，如果采用多层LSTM，那么就是下一层LSTM的delta_input。如果是LSTM接全连接这种，就只有最后一个LSTM单元的输出有梯度，其余为0。

接下来和公式推导过程一样，先计算ht的梯度，再计算Ct的梯度，最后计算权重和输入的梯度。delta_ht和delta_ct是后一个LSTM单元传给前一个单元的ht和Ct的梯度。本来需要判断是否是最后一个LSTM单元，如果是最后一个，那么delta_ht就等于输出方向传来的梯度，delta_ct就等于ht方向计算的梯度。其余的LSTM单元则是"+="，因为梯度来自两方面。但由于delta_ht和delta_ct本身就是零初始化的，所以全部采用"+="也没问题，还省去了判断过程。

在for循环结束后，也就是所有LSTM单元计算完权重的梯度后，再更新权重。

## 5 写在最后

{% note 注意权重初始化问题，当初我在实现的过程中，不太关注初始化，导致梯度消失很快。%}

{% note color:amber 第一次写博客，有问题欢迎在评论区提出 %}