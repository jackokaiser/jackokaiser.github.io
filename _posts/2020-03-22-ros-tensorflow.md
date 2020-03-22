---
title: 'ROS Tensorflow'
date: 2020-03-22
permalink: /posts/2020/03/ros-tensorflow
tags:
  - ROS
  - Tensorflow
  - Keras
---

In this blog post I go through some useful techniques to nicely integrate Tensorflow (TF) in ROS nodes.
The related code can be found in the following [ros_tensorflow](https://github.com/jackokaiser/ros_tensorflow_tutorial) github repo.

What this blog post covers:
- [Creating a TF model and saving the session](#model-creation)
- [Training a TF model in a ROS action](#model-training)
- [Making predictions from callbacks](#model-predictions)

Integrating Tensorflow 2 in ROS nodes
======

Tensorflow requires python 3, but ROS melodic and ubuntu 18.04 still rely on python 2 by default.
You can get away by running the Tensorflow node with python 3, using the shebang `#!/usr/bin/env python3`.
You will have to install some missing ROS python 3 libraries, which you can do with conda or pip.
If you plan to work with images, you will encounter an additional issue with cv\_bridge, which you can solve by following [this tutorial](https://medium.com/@beta_b0t/how-to-setup-ros-with-python-3-44a69ca36674).

Model creation
-----

Since `rospy` callbacks are called in a separate thread than the `main`, it is important to handle the Tensorflow `session` accordingly.
When you create your model at initialization phase in the `main` function, your code is running in the main thread.
Here, you should store the Tensorflow `session` so that you can restore it later in your callbacks:

```python
`#!/usr/bin/env python3`

import tensorflow as tf
import rospy

class ModelWrapper():
    def __init__(self):
        # store the session object from the main thread
        self.session = tf.compat.v1.keras.backend.get_session()

        self.model = tf.keras.Sequential()
        # ...

class RosInterface():
    def __init__(self):
        self.wrapped_model = ModelWrapper()

def main():
    rospy.init_node("ros_tensorflow")
    ri = RosInterface()
    rate = rospy.Rate(1)
    while not rospy.is_shutdown():
        rate.sleep()

if __name__ == "__main__":
    main()
```

Model training
-----

If you are recording data at runtime, you may want to re-train your model at runtime too.
The best way is to perform training in a ROS action server, so that a client can also cancel training if needed.
Since the action callback is called in a separate thread, you have to restore the session from the main thread.
Here is the training function for the tensorflow model:

```python
import tensorflow as tf

class ModelWrapper():
    # ...
    def train(self, x_train, y_train, n_epochs=100, callbacks=[]):
        with self.session.graph.as_default():
            tf.compat.v1.keras.backend.set_session(self.session)

            self.model.fit(x_train, y_train,
                epochs=n_epochs,
                callbacks=callbacks)
```

We will call this function from an action server.
Note that we declare a `callbacks` parameter, which will allow us to **provide action feedback** while the model is training and **cancel training** when the action is cancelled.
Here are some nice wrappers to achieve this:

```python
import tensorflow as tf

class StopTrainOnCancel(tf.keras.callbacks.Callback):
    def __init__(self, check_preempt):
        super(tf.keras.callbacks.Callback, self).__init__()
        self.check_preempt = check_preempt
    def on_batch_end(self, batch, logs={}):
        self.model.stop_training = self.check_preempt()

class EpochCallback(tf.keras.callbacks.Callback):
    def __init__(self, cb):
        super(tf.keras.callbacks.Callback, self).__init__()
        self.cb = cb
    def on_epoch_end(self, epoch, logs):
        self.cb(epoch, logs)
```

Now let's implement the ROS action server in our `RosInterface` class:

```python
import rospy
import actionlib
from ros_tensorflow_msgs.msg import TrainAction, TrainFeedback, TrainResult

class RosInterface():
    def __init__(self):
        self.wrapped_model = ModelWrapper()
        self.train_as = actionlib.SimpleActionServer('train', TrainAction,
                                                     self.train_cb, False)
        self.train_as.start()

    def train_cb(self, goal):
        stop_on_cancel = StopTrainOnCancel(
            check_preempt=lambda : self.train_as.is_preempt_requested())
        pub_feedback = EpochCallback(
            lambda epoch, logs: self.train_as.publish_feedback(
                TrainFeedback(i_epoch=epoch, loss=logs['loss'], acc=logs['accuracy'])))

        # ... load x_train and y_train
        self.wrapped_model.train(x_train, y_train, callbacks=[
            stop_on_cancel,
            pub_feedback])

        # Training finished either because it was done or because it was cancelled
        if self.train_as.is_preempt_requested():
            self.train_as.set_preempted()
        else:
            self.train_as.set_succeeded()
```

Model predictions
-----

There are two ways you can integrate a Tensorflow model in a ROS nodes:
- Making predictions in a loop
- Making predictions on callbacks

If you want to perform prediction on callbacks, you will have to restore the Tensorflow ```session``` of the main thread, just like training:

```python
class ModelWrapper():
    # ...
    def predict(self, x):
        with self.session.graph.as_default():
            tf.compat.v1.keras.backend.set_session(self.session)
            out = self.model.predict(x)
        return out
```

Restoring the session is not necessary if you perform predictions in the main loop, but it doesn't hurt.
Since predictions are fast compared to training, you can implement them with rostopic subscribers or rosservice servers, no need for actionlib in this case.

In the provided [ros_tensorflow](https://github.com/jackokaiser/ros_tensorflow_tutorial) repo I demonstrate both making predictions in a service callback and making predictions in main loop.
The repo also makes a clean separation between Tensorflow code and ROS code.
If you plan to use Tensorflow in your ROS project you can simply fork the repo and start from there.
