---
title:  “Create ONNX model using keras2onnx”
mathjax: true
layout: post
date:   2019-06-26 08:00:12 +0800
categories: deep learning
---

总结一下在用keras2onnx项目生成ONNX模型的流程以及一些问题。模型的生成不包含训练步骤，参数随机生成，因为最后仅仅需要验证graph的正确性。


# SimpleRNN/GRU/LSTM

## 1. SimpleRNN
```python
import numpy as np
import keras2onnx
import onnx
import keras

# batch_size = BAT
# seq_length = SEQ
# input_size = INP
# hidden_size = HID
BAT, SEQ, INP, HID = 3, 1, 2, 5

# initial state shape: (hidden_size, 1)
state = keras.Input(shape=(HID,))
# inputs shape: (seq_length, input_size)
inputs = keras.Input(shape=(SEQ, INP))

# create keras model
rnn_layer = keras.layers.SimpleRNN(HID, activation='relu', return_sequences=True)(inputs, initial_state=[state])
outputs = keras.layers.Dense(INP, activation='sigmoid')(rnn_layer)
keras_model = keras.Model(inputs=[inputs, state], outputs=outputs)

# convert to onnx
onnx_model = keras2onnx.convert_keras(keras_model, keras_model.name)

# Save
onnx.save_model(onnx_model, "keras-simple-rnn.onnx")

# predict
x = np.random.rand(BAT, SEQ, INP).astype(np.float32, copy=False)
s = np.random.rand(BAT, HID).astype(np.float32, copy=False)
expected = keras_model.predict([x, s])
```
![image01]({{site.baseurl}}/image/20190626/keras-simplernn.png)

## 2. GRU
```python
...
# create keras model
gru_layer = keras.layers.GRU(HID, activation='relu', return_sequences=True)(inputs, initial_state=[state])
outputs = keras.layers.Dense(INP, activation='sigmoid')(gru_layer)
keras_model = keras.Model(inputs=[inputs, state], outputs=outputs)

# convert to onnx
onnx_model = keras2onnx.convert_keras(keras_model, keras_model.name)

# Save
onnx.save_model(onnx_model, "keras-gru.onnx")
...
```
![image02]({{site.baseurl}}/image/20190626/keras-gru.png)

## 3. LSTM
```python
...
# initial state shape: (hidden_size, 1)
state_h = keras.Input(shape=(HID,))
state_c = keras.Input(shape=(HID,))

# inputs shape: (seq_length, input_size)
inputs = keras.Input(shape=(SEQ, INP))

# create keras model
lstm_layer = keras.layers.LSTM(units=HID, activation='relu', return_sequences=True)(inputs, initial_state=[state_h, state_c])
outputs = keras.layers.Dense(INP, activation='sigmoid')(lstm_layer)
keras_model = keras.Model(inputs=[inputs, state_h, state_c], outputs=outputs)

# convert to onnx
onnx_model = keras2onnx.convert_keras(keras_model, keras_model.name)

# Save
onnx.save_model(onnx_model, "keras-lstm.onnx")
...
```
![image03]({{site.baseurl}}/image/20190626/keras-lstm.png)

## keras2onnx.convert_keras转换失败

keras2onnx不支持`SimpleRNNCell/GRUCell/LSTMCell`。这个错误似乎是由tensorflow-onnx造成的[keras2onnx issue 115](https://github.com/onnx/keras-onnx/issues/115)。

```python
...

# create keras model
rnn_cell = keras.layers.SimpleRNNCell(C, activation='relu')
rnn_layer = keras.layers.RNN(rnn_cell)(inputs)
outputs = keras.layers.Dense(W, activation='sigmoid')(rnn_layer)
keras_model = keras.Model(inputs=inputs, outputs=outputs)

#  File "/Library/Frameworks/Python.framework/Versions/3.7/lib/python3.7/site-packages/keras2onnx-1.4.0-py3.7.egg/keras2onnx/ktf2onnx/tf2onnx/rewriter/loop_rewriter_base.py", line 230, in _check_in_read_only_mode
#    self._parse_loop_variables(context)
#  File "/Library/Frameworks/Python.framework/Versions/3.7/lib/python3.7/site-packages/keras2onnx-1.4.0-py3.7.egg/keras2onnx/ktf2onnx/tf2onnx/rewriter/loop_rewriter_base.py", line 244, in _parse_loop_variables
#    loop_var = self._get_loop_var_from_switch(s)
#  File "/Library/Frameworks/Python.framework/Versions/3.7/lib/python3.7/site-packages/keras2onnx-1.4.0-py3.7.egg/keras2onnx/ktf2onnx/tf2onnx/rewriter/loop_rewriter_base.py", line 391, in _get_loop_var_from_switch
#    ta_gather_node = [n for n in exit_consumers if is_tensor_array_gather_op(n)][0]
# IndexError: list index out of range
onnx_model = keras2onnx.convert_keras(keras_model, keras_model.name)
```

# 通过tensorflow转换

另外一种方式，我们将Keras模型先保存为一个pb文件，然后通过`tf2onnx.convert`命令转换为ONNX模型。下面以没有初始状态输入的SimpleRNN作为例子。

```python
import numpy as np
import tensorflow as tf
import keras

def freeze_session(session, keep_var_names=None, output_names=None, clear_devices=True):
    graph = session.graph
    with graph.as_default():
        freeze_var_names = list(set(v.op.name for v in tf.global_variables()).difference(keep_var_names or []))
        output_names = output_names or []
        output_names += [v.op.name for v in tf.global_variables()]
        input_graph_def = graph.as_graph_def()
        if clear_devices:
            for node in input_graph_def.node:
                node.device = ""
        frozen_graph = tf.graph_util.convert_variables_to_constants(
            session, input_graph_def, output_names, freeze_var_names)
        return frozen_graph

# batch_size = BAT
# seq_length = SEQ
# input_size = INP
# hidden_size = HID
BAT, SEQ, INP, HID = 3, 1, 2, 5

# inputs shape: (seq_length, input_size)
input = keras.Input(shape=(SEQ, INP), name="input_1")

# create keras model
rnn_layer = keras.layers.SimpleRNN(HID, activation='relu', return_sequences=True)(input)
outputs = keras.layers.Dense(INP, activation='sigmoid', name="output_1")(rnn_layer)
keras_model = keras.Model(inputs=input, outputs=outputs)

# Save as pb file
sess = keras.backend.get_session()
frozen_graph = freeze_session(sess, output_names=[out.op.name for out in keras_model.outputs])
tf.train.write_graph(frozen_graph, ".", "tf_simple_rnn.pb", as_text=False)

keras.backend.clear_session()
```
![image04]({{site.baseurl}}/image/20190626/tf-rnn-pb.png)

`python3 -m tf2onnx.convert --input ./tf_simple_rnn.pb --inputs input_1:0 --outputs output_1/Sigmoid:0 --output tf_simple_rnn.onnx --opset 8 --fold_const`将TensorFlow模型转换为ONNX模型。

![image05]({{site.baseurl}}/image/20190626/tf-rnn-onnx.png)