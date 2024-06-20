# **DL accelerator learning**

深度学习加速器的目的是将神经网络模型的inference过程部署在硬件平台上执行从而提高效率（降低延迟和资源消耗）

## **深度学习**

我们使用深度学习神经网络作为workload，不需要学习更深入的理论，所以只需要了解最重要的一些model，例如transformer，diffusion model，unet等，和一些重要的算子，例如卷积，GEMM，layernorm等，这些单独找资料理解就行。

[[Model\] Transformers 推导与细节笔记 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/696190481)

***\*侧重\****：神经网络模型分为training过程和inference过程，training并不是我们的重点，重点在inference，即用户给定model所需数据（例如gpt是用户输入一句话，stable diffusion是用户输入提示词让模型输出和提示词相关的图片），模型输出需要得到的数据，理解input如何一步一步转化为output是关键。理解转化过程可以分为理解关键组件，这块可以阅读model代码并跑一遍，[Hugging Face – The AI community building the future.](https://huggingface.co/) hugging face保存着基本所有所需的预训练模型，也就是可以直接使用并进行inference的model，使用方法可以参考一些教程，和关键算子，算子通过阅读PyTorch文档来了解功能（[PyTorch documentation — PyTorch 2.3 documentation](https://pytorch.org/docs/stable/index.html)）。

## **TVM**

tvm官方文档

[TVM 原理介绍 | Apache TVM 中文站 (hyper.ai)](https://tvm.hyper.ai/docs/tutorial/intro/)

通用深度学习编译器学习资料整理，辅助官网文档学习

[BBuf/tvm_mlir_learn: compiler learning resources collect. (github.com)](https://github.com/BBuf/tvm_mlir_learn)

陈天奇机器学习

[01 机器学习编译概述 【MLC-机器学习编译中文版】_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV15v4y1g7EU/?spm_id_from=333.337.search-card.all.click&vd_source=311729bf07a2e786419860a1d262bde5)

opu compiler code

[OPU-Lab/opu-compiler (github.com)](https://github.com/OPU-Lab/opu-compiler)



![DLcompiler](/Users/nineis/Documents/GitHub/DL-compiler-learning/asserts/pics/DLcompiler.png)





图 1 DL compiler

Translation为前端，Optimization为后端

我们编译器借助tvm实现前端模块，tvm将神经网络转化为计算图（一种tvm自定的IR(imme representation))，我们在计算图的基础上进行主要的Pass优化，例如算符融合（最重要）和一些编译器常规优化（DCE等）。一部分我们使用tvm自带的Pass（这一块读官方文档和资料去理解功能即可），另一块是自己编写的Pass，比如算符融合的Pass是我们自己写的算法，并没有使用tvm自己的算法，这一块可以通过阅读opu compiler code并实操去理解。

第二个重要的Pass是gen_ir，即生成第二种IR，该IR将前端总结的信息传入编译器后端，后端进行传统编译器通用优化方式来基于硬件的ISA生成硬件或平台所需的指令，指令的执行由硬件平台操作（RTL），与编译器无关。后端的操作有coarse-grained Instruction generation，Instruction Reordering，scheduling，Memory Allocation等，这些可以通过一些课程来学习理论，而对理论的实践可以参考compiler Backend code来学习。

***\*侧重\****：TVM是一套完整的通用深度学习编译器，我们只是使用其提供的接口去实现我们需要的功能，所以侧重于理解TVM的工作流和其中一些重要的优化就可以，但这也是很大的工作量，因为代码量很大。目前只需要理解opFusion和gen_ir的算法实现和其他一些算法的功能就行。这两个最主要Pass构成了大部分编译器Frontend的工作。

opFusion：算符融合，将一些访存密集型算子和一个计算密集型算子融合成一个算子的优化，我们对算子的执行分为***\*取数，计算，存数\****的过程，而取数和存数都是跨memory hierarchy的，例如对寄存器的访问速度最快，对DRAM的访问速度最慢，所以尽量减少数据和DRAM的交互是提高效率的重要手段，算符融合将访存密集型算子和计算密集型算子合并为一个算子，从而在计算后直接使用output而不需要将其存回DRAM。

gen_ir：生成给编译器后端使用的IR，并且存储下来硬件计算所需要的input，weight等数据。

 

## **Hardware for Machine Learning**

[加州大学伯克利分校 EE 290 机器学习硬件技术 Hardware for Machine Learning（Spring 2021）_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1hi4y1A7BC/?vd_source=311729bf07a2e786419860a1d262bde5)

该课程更多的侧重于compiler Backend领域，并且涉及到硬件知识，例如将kernel computation（即compute-intensive operator计算密集型算子）mapping到硬件（例如PE阵列，systolic array脉冲阵列等）是如何做的，如何在计算正确的基础上提高并行度，最大化利用资源和降低延迟，并且涉及到一些重要的研究方向，例如量化（浮点数转化为定点数，32位定点数转化为16位定点数等），稀疏（sparsity，大部分的计算都可能是x*0=0的无效计算，0的出现有可能因为量化的截断、relu算子的引入等，避免这些无效计算能够提高计算效率），最后提供big picture引入Software&Hardware co-design的工作模式，也就是通过设计编译器来生成指令流给硬件执行的方式。

 

## **Conclusion**

opu compiler的学习过程可以先理解compiler的input（即DL model），然后compiler的frontend从tvm入手理解，backend通过传统编译原理课程（[【硬核课程】编译器设计【使用教材：编译器设计(第2版)】_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Bp4y1t7Cw/?spm_id_from=333.1365.top_right_bar_window_custom_collection.content.click&vd_source=311729bf07a2e786419860a1d262bde5)侧重于register allocation，scheduling，instruction generation）和Hardware for Machine Learning等，前端和后端可以同步进行学习。Frontend的优化是hardware-independent，Backend是hardware-dependent，各自的侧重点不同。

 

书籍资料：

\- 编译器设计(第2版)

 