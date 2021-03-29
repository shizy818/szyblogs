---
title:  “Create ONNX model using tf2onnx"
mathjax: true
layout: post
date:   2019-06-27 08:00:12 +0800
categories: deep learning
---

接上一篇，tf2onnx项目生成ONNX模型的流程以及一些问题。


# tf2onnx.utils.save_onnx_model API

## 1. BasicRNNCell
```python
import numpy as np
import tf2onnx
import onnx
import tensorflow as tf
from tensorflow.python.ops import variables as variables_lib
from tf2onnx import utils
from tf2onnx.tfonnx import process_tf_graph, tf_optimize
from tf2onnx import optimizer

# batch_size = BAT
# seq_length = SEQ
# input_size = INP
# hidden_size = HID
BAT, SEQ, INP, HID = 3, 1, 2, 5

# input shape: (batch_size, seq_length, input_size) 
input = tf.placeholder(dtype=tf.float32, shape=(BAT, SEQ, INP), name="input_1")

# create tensorflow model
rnn_cell = tf.nn.rnn_cell.BasicRNNCell(num_units=HID)
rnn_out, _ = tf.nn.dynamic_rnn(rnn_cell, input, dtype=tf.float32)
output = tf.layers.dense(inputs=rnn_out, units=HID)
tf.identity(output, name="output_1")

input_names_with_port = ["input_1:0"]
output_names_with_port = ["output_1:0"]

x_val = np.random.rand(BAT, SEQ, INP).astype(np.float32, copy=False)

graph_def = None

# [[[-0.05846359 -0.06566401  0.02254938 -0.26033643 -0.07923548]]
# [[ 0.04879569  0.04215769 -0.06720451 -0.60583305  0.06223793]]
# [[-0.05626901 -0.06627436  0.00422506 -0.5533649  -0.0767431 ]]]
with tf.Session() as sess:
    # sess.run(tf.global_variables_initializer())
    variables_lib.global_variables_initializer().run()
    expected = sess.run(output, feed_dict={input : x_val})

    # convert variables to constants
    output_name_without_port = [n.split(':')[0] for n in output_names_with_port]
    graph_def = tf.graph_util.convert_variables_to_constants(sess, sess.graph_def, output_name_without_port)

tf.reset_default_graph()
tf.import_graph_def(graph_def, name='')

# [array([[[-0.05846359 -0.06566401  0.02254938 -0.26033643 -0.07923548]],
#         [[ 0.04879569  0.04215769 -0.06720451 -0.60583305  0.06223793]],
#         [[-0.05626901 -0.06627436  0.00422506 -0.5533649  -0.0767431 ]]], dtype=float32)]
with tf.Session() as sess:
    # output_dict: get tensor by output name
    output_dict = []
    for out_name in output_names_with_port:
        output_dict.append(sess.graph.get_tensor_by_name(out_name))

    expected = sess.run(output_dict, feed_dict={"input_1:0" : x_val})

# tf optimize
graph_def = tf_optimize(input_names_with_port, output_names_with_port, sess.graph_def, fold_constant=True)

tf.reset_default_graph()
tf.import_graph_def(graph_def, name='')

# convert to onnx
with tf.Session() as sess:
    g = process_tf_graph(sess.graph, output_names=output_names_with_port)
    g = optimizer.optimize_graph(g)
    model_proto = g.make_model("basic_rnn_cell")
    utils.save_onnx_model("./", "basic_rnn_cell", feed_dict={"input_1:0" : x_val}, model_proto=model_proto)
```
![image01]({{site.baseurl}}/image/20190627/tf-simplernn.png)

## 2. GRUCell
```python
...
# create tensorflow model
gru_cell = tf.nn.rnn_cell.GRUCell(num_units=HID)
gru_out, _ = tf.nn.dynamic_rnn(gru_cell, input, dtype=tf.float32)
output = tf.layers.dense(inputs=gru_out, units=HID)
tf.identity(output, name="output_1")

...

# convert to onnx
with tf.Session() as sess:
    g = process_tf_graph(sess.graph, output_names=output_names_with_port)
    g = optimizer.optimize_graph(g)
    model_proto = g.make_model("gru_cell")
    utils.save_onnx_model("./", "gru_cell", feed_dict={"input_1:0" : input}, model_proto=model_proto)
```
![image02]({{site.baseurl}}/image/20190627/tf-gru.png)

## 3. LSTMCell
```python
...
# create tensorflow model
lstm_cell = tf.nn.rnn_cell.LSTMCell(num_units=HID)
lstm_out, _ = tf.nn.dynamic_rnn(lstm_cell, input, dtype=tf.float32)
output = tf.layers.dense(inputs=lstm_out, units=HID)
tf.identity(output, name="output_1")

...

# convert to onnx
with tf.Session() as sess:
    g = process_tf_graph(sess.graph, output_names=output_names_with_port)
    g = optimizer.optimize_graph(g)
    model_proto = g.make_model("lstm_cell")
    utils.save_onnx_model("./", "lstm_cell", feed_dict={"input_1:0" : input}, model_proto=model_proto)
```
![image03]({{site.baseurl}}/image/20190627/tf-lstm.png)

## 4. MultiRNNCell (BasicRNNCell/GRUCell/LSTMCell)

对于`BasicRNNCell/GRUCell/LSTMCell`，tf2onnx将`MutiRNNCell`转换为`Scan Operator`。

```python
...
# create 2 BasicRNNCell
rnn_layers = [tf.nn.rnn_cell.BasicRNNCell(size) for size in [HID, HID]]

# create a RNN cell composed sequentially of a number of RNNCells
multi_rnn_cell = tf.nn.rnn_cell.MultiRNNCell(rnn_layers)
rnn_out, _ = tf.nn.dynamic_rnn(cell=multi_rnn_cell, inputs=input, dtype=tf.float32)
output = tf.layers.dense(inputs=rnn_out, units=HID)
tf.identity(output, name="output_1")

...
```
![image04]({{site.baseurl}}/image/20190627/multi-basic-rnn.png)

# tf2onnx.convert命令

另外一种方式，我们将Tensorflow模型先保存为一个pb文件，然后通过`tf2onnx.convert`命令转换为ONNX模型。此时tf2onnx也引入了`Scan Operator`。

```python
import numpy as np
import tensorflow as tf
from tensorflow.python.ops import variables as variables_lib

# batch_size = BAT
# seq_length = SEQ
# input_size = INP
# hidden_size = HID
BAT, SEQ, INP, HID = 3, 1, 2, 5

# input shape: (batch_size, seq_length, input_size) 
input = tf.placeholder(dtype=tf.float32, shape=(BAT, SEQ, INP), name="input_1")

# create tensorflow model
rnn_cell = tf.nn.rnn_cell.BasicRNNCell(num_units=HID)
rnn_out, _ = tf.nn.dynamic_rnn(rnn_cell, input, dtype=tf.float32)
output = tf.layers.dense(inputs=rnn_out, units=HID)
tf.identity(output, name="output_1")

input_names_with_port = ["input_1:0"]
output_names_with_port = ["output_1:0"]

# convert variables to constants
with tf.Session() as sess:
    variables_lib.global_variables_initializer().run()
    output_name_without_port = [n.split(':')[0] for n in output_names_with_port]
    graph_def = tf.graph_util.convert_variables_to_constants(sess, sess.graph_def, output_name_without_port)
    tf.train.write_graph(graph_def, '.', 'basic_rnn_cell.pb', as_text=False)
```
![image05]({{site.baseurl}}/image/20190627/basic-rnn-pb.png)

`python3 -m tf2onnx.convert --input ./basic_rnn_cell.pb --inputs input_1:0 --outputs output_1:0 --output basic_rnn_cell.onnx --opset 8 --fold_const`将TensorFlow模型转换为ONNX模型。

![image06]({{site.baseurl}}/image/20190627/basic-rnn-onnx.png)

- tf2onnx.convert简略分析

基本上tf2onnx.convert包括几个步骤：
* `tf_optimize`：优化tensorflow graph，比如fold_const
* `process_tf_graph`： 将tensorflow graph转换为ONNX graph
* `make_model`：通过ONNX graph创建最终的ONNX ModelProto
* `optimize_model_proto`：优化ONNX ModelProto，比如去除所有无用的Transpose

```python
def main():
...

    # todo: consider to enable const folding by default?
    graph_def = tf_optimize(inputs, outputs, graph_def, args.fold_const)

    with tf.Graph().as_default() as tf_graph:
        tf.import_graph_def(graph_def, name='')
    with tf.Session(graph=tf_graph):
        g = process_tf_graph(tf_graph,
                             continue_on_error=args.continue_on_error,
                             verbose=args.verbose,
                             target=args.target,
                             opset=args.opset,
                             custom_op_handlers=custom_ops,
                             extra_opset=extra_opset,
                             shape_override=args.shape_override,
                             input_names=inputs,
                             output_names=outputs,
                             inputs_as_nchw=args.inputs_as_nchw)

    model_proto = g.make_model("converted from {}".format(model_path))

    new_model_proto = GraphUtil.optimize_model_proto(model_proto)
    if new_model_proto:
        model_proto = new_model_proto
    else:
        print("NON-CRITICAL, optimizers are not applied successfully")

    # write onnx graph
    if args.output:
        utils.save_protobuf(args.output, model_proto)
        print("\nComplete successfully, the onnx model is generated at " + args.output)
```

# onnx-experiments.py脚本

在tools目录下有一个脚本onnx-experiments.py，用于`rewrite_constant_fold`，其作用的Operator包括Transpose，Unsqueeze，Slice，Add，Cast，Mul，Sqrt和Sub。

```python
def rewrite_constant_fold(g, ops):

    func_map = {
        "Transpose": not None,
        "Unsqueeze": not None,
        "Slice": not None,
        "Add": np.add,
        "Cast": np.cast,
        "Mul": np.multiply,
        "Sqrt": np.sqrt,
        "Sub": np.subtract,
    }

    # pylint: disable=too-many-nested-blocks
    keep_looking = True
    while keep_looking:
        keep_looking = False
        for idx, op in enumerate(ops):
            inputs = []
            for node in op.inputs:
                if node and node.is_const():
                    inputs.append(node.get_tensor_value(as_list=False))

            if inputs and len(op.input) == len(inputs):
                func = func_map.get(op.type)
                if func is None:
                    log.info("can fold but don't know how, type=%s, name=%s", op.type, op.name)
                    continue
                try:
                    log.info("folding node type=%s, name=%s", op.type, op.name)
                    if op.type == "Cast":
                        dst = op.get_attr_int("to")
                        np_type = tf2onnx.utils.map_onnx_to_numpy_type(dst)
                        val = np.cast[np_type](*inputs)
                    elif op.type == "Transpose":
                        perm = op.get_attr("perm").ints
                        val = np.transpose(inputs[0], perm)
                    elif op.type == "Unsqueeze":
                        axis = op.get_attr_int("axis")
                        val = np.expand_dims(inputs[0], axis=axis)
                    elif op.type == "Slice":
                        axis = op.get_attr_int("axis")
                        if axis != 0:
                            log.info("can fold slice with axis!=0, type=%s, name=%s", op.type, op.name)
                            continue
                        starts = op.get_attr_int("starts")
                        ends = op.get_attr_int("ends")
                        if starts == 0 and ends == 0:
                            val = inputs[0][starts:ends]
                        else:
                            val = inputs[0]
                    else:
                        val = func(*inputs)

                    new_node_name = tf2onnx.utils.make_name(op.name)
                    new_output_name = new_node_name
                    old_output_name = op.output[0]
                    old_node_name = op.name
                    log.debug("create const node [%s] replacing [%s]", new_node_name, old_node_name)
                    ops[idx] = g.make_const(new_node_name, val)
                    consumers = g.find_output_consumers(old_output_name)
                    if consumers:
                        for consumer in consumers:
                            g.replace_input(consumer, old_output_name, new_output_name)
                    for i, node in zip(op.input, op.inputs):
                        if len(g.find_output_consumers(i)) == 1:
                            g.remove_node(node.name)
                    keep_looking = True
                except Exception as ex:  # pylint: disable=broad-except
                    tb = traceback.format_exc()
                    log.info("exception: %s, details: %s", ex, tb)
                    # pylint: enable=too-many-nested-blocks
    return ops
```

下面用Reshape的例子来具体说明onnx-experiments.py的作用。

```python
import numpy as np
import tensorflow as tf

x_input = np.array([[[1, 1, 1], [2, 2, 2]],
                 [[3, 3, 3], [4, 4, 4]],
                 [[5, 5, 5], [6, 6, 6]]])

x = tf.placeholder(dtype=tf.float32, shape=(3, 2, 3), name='x')
s = tf.cast([2, 3, 3], dtype = tf.int64)
y = tf.reshape(x, s)
tf.identity(y, name='y')

with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    # print(sess.run(y, feed_dict={x : x_input}))

    graph_def = tf.graph_util.convert_variables_to_constants(
        sess,
        sess.graph.as_graph_def(add_shapes=True),
        ['y'],
    )
    tf.train.write_graph(graph_def, '.', 'reshape.pb', as_text=False)
```
![image07]({{site.baseurl}}/image/20190627/reshape-pb.png)

然后运行命令`python3 -m tf2onnx.convert --input reshape.pb --output reshape.onnx --opset 8 --inputs x:0 --outputs y:0 --fold_const`将Tensorflow模型转换为ONNX模型。

![image08]({{site.baseurl}}/image/20190627/reshape-onnx.png)

然而`Cast Operator`其实是多余的，因为tensor`[2, 3, 3]`作为Reshape的输入是一个常数，可以将其移入Reshape的`shape initializer`。运行`python3 onnx-experiments.py --input reshape.onnx --output reshape-initializer.onnx`，结果如下：

![image09]({{site.baseurl}}/image/20190627/reshape-onnx-init.png)

在最新的tf2onnx项目里，似乎rewrite_constant_fold已经是优化默认行为，不需要单独调用了。