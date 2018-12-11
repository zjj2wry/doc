### 支持的设备
在一套标准系统中通常有多台计算设备。TensorFlow 支持 CPU 和 GPU 这两种设备。它们均用 strings 表示。例如：
"/cpu:0"：机器的 CPU
"/device:GPU:0"：机器的 GPU（如果有一个）
"/device:GPU:1"：机器的第二个 GPU（以此类推）

如果 TensorFlow 指令中兼有 CPU 和 GPU 实现，当该指令分配到设备时，GPU 设备有优先权。例如，如果 matmul 同时存在 CPU 和 GPU 核函数，在同时有 cpu:0 和 gpu:0 设备的系统中，gpu:0 会被选来运行 matmul。

### 记录设备分配方式
要找出您的指令和张量被分配到哪个设备，请创建会话并将 log_device_placement 配置选项设为 True。
```python
import tensorflow as tf
# Creates a graph.
a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
c = tf.matmul(a, b)
# Creates a session with log_device_placement set to True.
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
# Runs the op.
print(sess.run(c))
```
您应该会看到以下输出内容：
```bash
Device mapping:
/job:localhost/replica:0/task:0/device:GPU:0 -> device: 0, name: Tesla K40c, pci bus
id: 0000:05:00.0
b: /job:localhost/replica:0/task:0/device:GPU:0
a: /job:localhost/replica:0/task:0/device:GPU:0
MatMul: /job:localhost/replica:0/task:0/device:GPU:0
[[ 22.  28.]
 [ 49.  64.]]
```

### 手动分配设备
如果您希望特定指令在您选择的设备（而非系统自动为您选择的设备）上运行，您可以使用 with tf.device 创建设备上下文，这个上下文中的所有指令都将被分配在同一个设备上运行。
```python
import tensorflow as tf
# Creates a graph.
with tf.device('/cpu:0'):
  a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
  b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
c = tf.matmul(a, b)
# Creates a session with log_device_placement set to True.
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
# Runs the op.
print(sess.run(c))
```

您会看到现在 a 和 b 被分配到 cpu:0。由于未明确指定运行 MatMul 指令的设备，因此 TensorFlow 运行时将根据指令和可用设备（此示例中的 gpu:0）选择一个设备，并会根据要求自动复制设备间的张量。
```bash
Device mapping:
/job:localhost/replica:0/task:0/device:GPU:0 -> device: 0, name: Tesla K40c, pci bus
id: 0000:05:00.0
b: /job:localhost/replica:0/task:0/cpu:0
a: /job:localhost/replica:0/task:0/cpu:0
MatMul: /job:localhost/replica:0/task:0/device:GPU:0
[[ 22.  28.]
 [ 49.  64.]]
```


### 允许增加 GPU 内存
默认情况下，TensorFlow 会映射进程可见的所有 GPU 的几乎所有 GPU 内存（取决于 CUDA_VISIBLE_DEVICES）。通过减少内存碎片，可以更有效地使用设备上相对宝贵的 GPU 内存资源。

在某些情况下，最理想的是进程只分配可用内存的一个子集，或者仅根据进程需要增加内存使用量。TensorFlow 在 Session 上提供两个 Config 选项来进行控制。

第一个是 allow_growth 选项，它试图根据运行时的需要来分配 GPU 内存：它刚开始分配很少的内存，随着 Session 开始运行并需要更多 GPU 内存，我们会扩展 TensorFlow 进程所需的 GPU 内存区域。请注意，我们不会释放内存，因为这可能导致出现更严重的内存碎片情况。要开启此选项，请通过以下方式在 ConfigProto 中设置选项：
config = tf.ConfigProto()
config.gpu_options.allow_growth = True
session = tf.Session(config=config, ...)

如要真正限制 TensorFlow 进程可使用的 GPU 内存量，这非常实用。


### 在多 GPU 系统中使用单一 GPU
如果您的系统中有多个 GPU，则默认情况下将选择 ID 最小的 GPU。如果您希望在其他 GPU 上运行，则需要显式指定偏好设置：
```python
import tensorflow as tf
# Creates a graph.
with tf.device('/device:GPU:2'):
  a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
  b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
  c = tf.matmul(a, b)
# Creates a session with log_device_placement set to True.
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
# Runs the op.
print(sess.run(c))
```python

如果您指定的设备不存在，您会看到 InvalidArgumentError：
InvalidArgumentError: Invalid argument: Cannot assign a device to node 'b':
Could not satisfy explicit device specification '/device:GPU:2'
   [[Node: b = Const[dtype=DT_FLOAT, value=Tensor<type: float shape: [3,2]
   values: 1 2 3...>, _device="/device:GPU:2"]()]]

当指定设备不存在时，如果您希望 TensorFlow 自动选择现有的受支持设备来运行指令，则可以在创建会话时将配置选项中的 allow_soft_placement 设为 True。
```python
import tensorflow as tf
# Creates a graph.
with tf.device('/device:GPU:2'):
  a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
  b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
  c = tf.matmul(a, b)
# Creates a session with allow_soft_placement and log_device_placement set
# to True.
sess = tf.Session(config=tf.ConfigProto(
      allow_soft_placement=True, log_device_placement=True))
# Runs the op.
print(sess.run(c))
```

使用多个 GPU
如果您想要在多个 GPU 上运行 TensorFlow，则可以采用多塔式方式构建模型，其中每个塔都会分配给不同 GPU。例如：
```python
import tensorflow as tf
# Creates a graph.
c = []
for d in ['/device:GPU:2', '/device:GPU:3']:
  with tf.device(d):
    a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3])
    b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2])
    c.append(tf.matmul(a, b))
with tf.device('/cpu:0'):
  sum = tf.add_n(c)
# Creates a session with log_device_placement set to True.
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
# Runs the op.
print(sess.run(sum))
```

您会看到以下输出内容：
```bash
Device mapping:
/job:localhost/replica:0/task:0/device:GPU:0 -> device: 0, name: Tesla K20m, pci bus
id: 0000:02:00.0
/job:localhost/replica:0/task:0/device:GPU:1 -> device: 1, name: Tesla K20m, pci bus
id: 0000:03:00.0
/job:localhost/replica:0/task:0/device:GPU:2 -> device: 2, name: Tesla K20m, pci bus
id: 0000:83:00.0
/job:localhost/replica:0/task:0/device:GPU:3 -> device: 3, name: Tesla K20m, pci bus
id: 0000:84:00.0
Const_3: /job:localhost/replica:0/task:0/device:GPU:3
Const_2: /job:localhost/replica:0/task:0/device:GPU:3
MatMul_1: /job:localhost/replica:0/task:0/device:GPU:3
Const_1: /job:localhost/replica:0/task:0/device:GPU:2
Const: /job:localhost/replica:0/task:0/device:GPU:2
MatMul: /job:localhost/replica:0/task:0/device:GPU:2
AddN: /job:localhost/replica:0/task:0/cpu:0
[[  44.   56.]
 [  98.  128.]]
```

cifar10 教程 就是个很好的例子（https://tensorflow.google.cn/tutorials/images/deep_cnn?hl=zh-CN），演示了如何使用多个 GPU 进行训练。
