### CMU eric xing

当前问题 数据大，参数多，需要分布式的计算

举了一个例子

ml computation vs classical 
traditional ：operation-centric and deterministic
ML :optimization-centric and iterative convergent

properties

算法和系统重新设计，互相适应

分布式的问题：怎么分，用什么做桥梁，怎么交流，交流什么

**avoid dependency errors via structure aware parallelization(SAP）**

### google brain

1. programming model evolution
   不是越快越好，要保证精确度
   
2. hardware 	platforms
   
  TPU tensor processing unit2
  
  model parrallelism om tpus is great!
  
3. model architectures
   
4. ML inside the system

solution = data + computation + ML expertise


### berkeley. intersection of ai+system

发展历程  intergratoin 

big data vs application (decision query)

support low-latency,high-throughput serving workloads

wide range of application snad framework

predicition serving is an open problems

batching to improve throughput 

model costs are increaing fast than 精确度


### TVM
computation graph

TVM :low level IR

https://github.com/dmlc/tvm

https://github.com/dmlc/nnvm


### chainer MN

 a flexible deep learning framework
 
### easyML  ease the process of ml with dataflow (dataflow 计算所)

key features :   resource managment   task design.   task monitoring.  task reusing 
 
### microsoft. 
让app具有人的智慧

### nvidia
CUDA & CUDA libraries

communication libraries. NCCL. MPI

frameworks. caffe ...

simulation ai任务与real word 联系紧密，所以需要simulation   nvidia project isaac ：simulation for RL

deep neural networks. 见图片

scale matters  more data more ai

laws of physics. successful ai uses accelerated  computing.  gpu is accelerator ai is huge focus for gpu

tensor core 

scalability

hardware platforms. not just gpus  driver px pegasus、DGX

tensor RT  见图片

### DAWN 
























