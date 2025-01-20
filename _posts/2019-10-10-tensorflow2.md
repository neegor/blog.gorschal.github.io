---
layout: post
title: Что нового в TensorFlow 2.0
categories:
  - Разработка
  - Develop
tags:
  - develop
  - tensorflow
  - mashinelerning
  - ai
---

**TensorFlow** как никто другой оказывает влияние на **machine learning** комьюнити. Фреймворк с недавних пор уже в бете с литерой 2.0. Это достаточно серьезный апдейт и в этом тексте хочу осветить наиболее значимые моменты.

Это не руководство по **AI** и для понимания вы уже должны быть знакомы с **TensorFlow**.

## Установка и демонстрационный набор данных

Используем Jupyter Notebook:

```bash
$ pip install tensorflow==2.0.0-beta1
```

Если необходимо использовать графический процессор (CUDA):

```bash
$ pip install tensorflow-gpu==2.0.0-beta1
```

Нам также будет необходим тестовый набор данных. Будем использовать [набор данных Adult](https://archive.ics.uci.edu/ml/datasets/adult) из архива UCI.

```python
import pandas as pd

columns = ["Age", "WorkClass", "fnlwgt", "Education", "EducationNum",
        "MaritalStatus", "Occupation", "Relationship", "Race", "Gender",
        "CapitalGain", "CapitalLoss", "HoursPerWeek", "NativeCountry", "Income"]

data = pd.read_csv('https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.data',
                    header=None,
                    names=columns)

data.head()
```

Набор данных представляет собой задачу двоичной классификации, которая заключается в прогнозировании, будет ли человек зарабатывать более 50 000 долларов в год.

Для начала проведем базовую предварительную обработку данных, а затем настроим разделение данных в соотношении 80:20:

```python
from sklearn.preprocessing import LabelEncoder
from sklearn.model_selection import train_test_split
import numpy as np

# Label Encode
le = LabelEncoder()
data = data.apply(le.fit_transform)

# Segregate data features & convert into NumPy arrays
X = data.iloc[:, 0:-1].values
y = data['Income'].values

# Split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=7)
```

Если **TensorFlow** у нас установился, датасет загрузился, то можно переходить к самому интересному.

## 1. Eager execution по умолчанию

В **TensorFlow 2.0** больше не нужно создавать сессию и запускать в нем граф вычислений. [Eager execution](https://www.tensorflow.org/alpha/guide/eager) включено по умолчанию в версии 2.0, так что вы можете создавать модели и запускать их мгновенно. Отключить *eager execution* можно следующим образом:

```tf.compat.v1.disable_eager_execution()``` (provided ```tensorflow``` is imported with ```tf``` alias.)

## 2. tf.function и AutoGraph

В то время как eager execution позволяет вам работать с императивным программированием, но как только речь заходит о распределенном обучении, полномасштабной оптимизации, производственные среды *style graph execution* **TensorFlow 1.x** имеет свои преимущества по сравнению с *eager execution*. В **TensorFlow 2.0** вы сохраняете выполнение на основе графов, но более гибким способом. Это можно сделать с помощью [tf.function](https://www.tensorflow.org/alpha/tutorials/eager/tf_function) и [AutoGraph](https://www.tensorflow.org/alpha/guide/autograph).

*tf.function* позволяет определять графы **TensorFlow** с синтаксисом в стиле Python с помощью функции *AutoGraph*. *AutoGraph* поддерживает широкий диапазон совместимости с Python, включая оператор *if*, цикл *for*, цикл *while*, *итераторы* и т. д. Однако существуют ограничения. [Здесь](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/autograph/LIMITATIONS.md#python-language-support-status) вы можете ознакомиться с полным списком доступных на данный момент операторов. Ниже приведен пример, показывающий, как легко определить граф **TensorFlow** с помощью простого *декоратора*.

```python
import tensorflow as tf

# Define the forward pass
@tf.function
def single_layer(x, y):
    return tf.nn.relu(tf.matmul(x, y))

# Generate random data drawn from a uniform distribution
x = tf.random.uniform((2, 3))
y = tf.random.uniform((3, 5))

single_layer(x, y)
```

Обратите внимание, что вам не нужно было создавать сессии или поля для заполнения для запуска функции ```single_layer ()```. Это одна из замечательных функций *tf.function*. Из-за этого он выполняет все необходимые оптимизации, чтобы код работал быстрее.

## 3. tf.variable_scope больше не нужен

В версии **TensorFlow 1.x**, для того, чтобы использовать *tf.layers* в качестве переменных и использовать их повторно, Вам нужно был использовать блок *tf.variable*. Но этого больше не нужно делать в **TensorFlow 2.0**. Из-за присутствия keras в качестве центрального высокоуровневого API в **TensorFlow 2.0** все слои, созданные с использованием *tf.layers*, могут быть легко помещены в определение *tf.keras.Sequential*. Это значительно облегчает чтение кода, а также позволяет отслеживать переменные и потери.

Пример:

```python
# Define the model
model = tf.keras.Sequential([
    tf.keras.layers.Dropout(rate=0.2, input_shape=X_train.shape[1:]),
    tf.keras.layers.Dense(units=64, activation='relu'),
    tf.keras.layers.Dropout(rate=0.2),
    tf.keras.layers.Dense(units=64, activation='relu'),
    tf.keras.layers.Dropout(rate=0.2),
    tf.keras.layers.Dense(units=1, activation='sigmoid')
])

# Get the output probabilities
out_probs = model(X_train.astype(np.float32), training=True)
print(out_probs)
```

В приведенном выше примере вы передали набор данных для обучения через *model* просто для получения необработанных выходных вероятностей. Обратите внимание, что это просто прямой проход. Вы можете, конечно, пойти вперед и обучать свою модель:

```python
model.compile(loss='binary_crossentropy', optimizer='adam')

model.fit(X_train, y_train,
              validation_data=(X_test, y_test),
              epochs=5, batch_size=64)
```

Вы можете получить список обучаемых параметров модели по слоям, например:

```python
# Model's trainable parameters in a layer by layer fashion
model.trainable_variables
```

## 4. Кастомные слои это просто

В исследованиях в области машинного обучения и в промышленных приложениях часто возникает необходимость в написании кастомных слоев для удовлетворения конкретных случаев использования. **TensorFlow 2.0** позволяет очень просто написать собственный слой и использовать его вместе с существующими слоями. Вы также можете настроить прямой проход вашей модели любым удобным для вас способом.

Чтобы создать кастомный слой, проще всего расширить класс *Layer* из *tf.keras.layers*, а затем определить его соответствующим образом. Вы создадите кастомный слой, а затем определите его прямые вычисления.

Можно создать несколько слоев, расширяя класс *Model* из *tf.keras*. Подробнее о создании моделей можно узнать [здесь](https://www.tensorflow.org/beta/tutorials/eager/custom_layers#models_composing_layers).

## 5. Гибкость в обучении модели

**TensorFlow** может использовать [автоматическое дифференцирование](https://www.tensorflow.org/alpha/tutorials/eager/automatic_differentiation) для вычисления градиентов функции потерь относительно параметров модели. *tf.GradientTape* создает ленту в контексте, который используется **TensorFlow** для отслеживания градиентов, записанных при каждом вычислении на этой ленте. Чтобы понять это, давайте определим модель на более низком уровне, расширив класс *tf.keras.Model*.

```python
from tensorflow.keras import Model

class CustomModel(Model):
    def __init__(self):
        super(CustomModel, self).__init__()
        self.do1 = tf.keras.layers.Dropout(rate=0.2, input_shape=(14,))
        self.fc1 = tf.keras.layers.Dense(units=64, activation='relu')
        self.do2 = tf.keras.layers.Dropout(rate=0.2)
        self.fc2 = tf.keras.layers.Dense(units=64, activation='relu')
        self.do3 = tf.keras.layers.Dropout(rate=0.2)
        self.out = tf.keras.layers.Dense(units=1, activation='sigmoid')

    def call(self, x):
        x = self.do1(x)
        x = self.fc1(x)
        x = self.do2(x)
        x = self.fc2(x)
        x = self.do3(x)
        return self.out(x)

model = CustomModel()
```

Обратите внимание, что топология этой модели точно такая же, как и та, которую вы определили ранее. Чтобы обучить эту модель с использованием автоматического дифференцирования, вам нужно по-разному определить функцию потерь и оптимайзер:

```python
loss_func = tf.keras.losses.BinaryCrossentropy()
optimizer = tf.keras.optimizers.Adam()
```

Теперь вы определите метрики, которые будут использоваться для измерения производительности сети, включающей ее обучение. Под производительностью здесь подразумеваются потеря и точность модели.

```python
# Average the loss across the batch size within an epoch
train_loss = tf.keras.metrics.Mean(name='train_loss')
train_acc = tf.keras.metrics.BinaryAccuracy(name='train_acc')

valid_loss = tf.keras.metrics.Mean(name='test_loss')
valid_acc = tf.keras.metrics.BinaryAccuracy(name='valid_acc')
```

*tf.data* предоставляет служебные методы для определения конвейеров входных данных. Это особенно полезно, когда вы имеете дело с большим объемом данных.

Теперь вы определите генератор данных, который будет генерировать пакеты данных во время обучения модели.

```python
X_train, X_test = X_train.astype(np.float32), X_test.astype(np.float32)
y_train, y_test = y_train.astype(np.int64), y_test.astype(np.int64)
y_train, y_test = y_train.reshape(-1, 1), y_test.reshape(-1, 1)

# Batches of 64
train_ds = tf.data.Dataset.from_tensor_slices((X_train, y_train)).batch(64)
test_ds = tf.data.Dataset.from_tensor_slices((X_test, y_test)).batch(64)
```

Теперь вы готовы обучать модель, используя *tf.GradientTape*. Во-первых, вы определите метод, который будет тренировать модель с данными, которые вы только что определили, используя *tf.data.DataSet*. Вы также оберните этапы обучения модели с помощью декоратора *tf.function*, чтобы ускорить вычисления.

## 6. Наборы данных TensorFlow

Отдельный модуль *DataSets* используется для отточенной работы с сетевой моделью. Вы уже видели это в предыдущем примере. В этом разделе вы увидите, как можно загрузить набор данных MNIST именно так, как вам нужно.

Вы можете установить библиотеку *tenorflow_datasets* с помощью *pip*. Как только он установлен, вы готовы приступать к работе. Он предоставляет несколько вспомогательных функций, которые помогут вам гибко подготовить конвейер построения набора данных. Вы можете узнать больше об этих функциях [здесь](https://github.com/tensorflow/datasets) и [здесь](https://www.youtube.com/watch?v=-nTe44WT0ZI). Теперь вы увидите, как вы можете построить конвейер ввода данных для загрузки в наборе данных MNIST.

```python
import tensorflow_datasets as tfds

# You can fetch the DatasetBuilder class by string
mnist_builder = tfds.builder("mnist")

# Download the dataset
mnist_builder.download_and_prepare()

# Construct a tf.data.Dataset: train and test
ds_train, ds_test = mnist_builder.as_dataset(split=[tfds.Split.TRAIN, tfds.Split.TEST])
```

Вы можете игнорировать предупреждение. Обратите внимание, как оригинально *tennsflow_datasets* обрабатывает конвейер.

```python
# Prepare batches of 128 from the training set
ds_train = ds_train.batch(128)

# Load in the dataset in the simplest way possible
for features in ds_train:
    image, label = features["image"], features["label"]
```

Теперь вы можете отобразить первое изображение из коллекции изображений, в которую вы загрузили. Обратите внимание, что *tennsflow_datasets* работает в активном режиме и настроен на основе графа.

```python
import matplotlib.pyplot as plt
%matplotlib inline

# You can convert a TensorFlow tensor just by using
# .numpy()
plt.imshow(image[0].numpy().reshape(28, 28), cmap=plt.cm.binary)
plt.show()
```
## 7. Политика автоматической смешанной точности

Политика смешанной точности была предложена NVIDIA в прошлом году. Оригинал статьи находится [здесь](https://arxiv.org/abs/1710.03740). Кратко об основной идеи политики смешанной точности: использовать смесь половин (FP16) и полной точности (FP32) и использовать преимущества обоих миров. Она показала потрясающие результаты в обучении очень глубоких нейронных сетей (как с точки зрения времени и оценки).

Если вы работаете в среде графического процессора с поддержкой CUDA (например, Volta Generation, Tesla T4) и установили вариант **TensorFlow 2.0** с графическим процессором, вы можете дать **TensorFlow** команду обучаться со смешанной точностью, например, так:

```os.environ['TF_ENABLE_AUTO_MIXED_PRECISION'] = '1'```

Это автоматически распределит операции графа TensorFlow соответственно. Вы сможете увидеть хороший скачок производительности вашей модели. Вы также можете оптимизировать основные операции TensorFlow с помощью политики смешанной точности. Подробнее почитайте в этой [статье](https://docs.nvidia.com/deeplearning/sdk/mixed-precision-training/index.html).

## 8. Распределенное обучение

**TensorFlow 2.0** позволяет очень легко распределять процесс обучения по нескольким графическим процессорам. Это особенно полезно для производственных целей, когда вам приходится встречать сверхтяжелые нагрузки. Это так же просто, как поместить ваш тренировочный блок в блок *with*.

Сначала вы указываете стратегию распределения следующим образом:

```mirrored_strategy = tf.distribute.MirroredStrategy()```

Зеркальная стратегия создает одну реплику для каждого графического процессора, а переменные модели одинаково отражаются в разных графических процессорах. Теперь вы можете использовать определенную стратегию следующим образом:

```python
with mirrored_strategy.scope():
    model = tf.keras.Sequential([tf.keras.layers.Dense(1, input_shape=(1,))])
    model.compile(loss='mse', optimizer='sgd')
    model.fit(X_train, y_train,
             validation_data=(X_test, y_test),
             batch_size=128,
             epochs=10)
```

**Обратите внимание**, что приведенный выше фрагмент кода будет полезен только в том случае, если в одной системе настроено несколько графических процессоров. Существует несколько стратегий распределения, которые вы можете настроить. Подробнее об этом можно прочитать [здесь](https://www.tensorflow.org/alpha/guide/distribute_strategy).

## 9. TensorBoard в Jupyter notebook

Это, пожалуй, самая захватывающая часть этого апдейта. Вы можете визуализировать обучение модели непосредственно в **Jupyter notebook** через *TensorBoard*. Новый *TensorBoard* имеет множество интересных функций, таких как профилирование памяти, просмотр данных изображения, включая матрицу неточностей, граф концептуальной модели и так далее. Подробнее об этом можно прочитать [здесь](https://www.tensorflow.org/tensorboard/r2/get_started).

В этом разделе вы сконфигурируете свою среду так, чтобы *TensorBoard* отображался в **Jupyter**. Сначала вам нужно загрузить расширение блокнота *tenorboard.notebook*:

```bash
%load_ext tensorboard.notebook
```

Теперь вы определите обратный вызов *TensorBoard* с помощью модуля *tf.keras.callbacks*.

```python
from datetime import datetime
import os

# Make a directory to keep the training logs
os.mkdir("logs")

# Set the callback
logdir = "logs"
tensorboard_callback = tf.keras.callbacks.TensorBoard(log_dir=logdir)
```

Перестройте модель, используя *Sequential* API *tf.keras*:

```python
# Define the model
model = tf.keras.Sequential([
    tf.keras.layers.Dropout(rate=0.2, input_shape=X_train.shape[1:]),
    tf.keras.layers.Dense(units=64, activation='relu'),
    tf.keras.layers.Dropout(rate=0.2),
    tf.keras.layers.Dense(units=64, activation='relu'),
    tf.keras.layers.Dropout(rate=0.2),
    tf.keras.layers.Dense(units=1, activation='sigmoid')
])

# Compile the model
model.compile(loss='binary_crossentropy', optimizer='adam', metrics=['accuracy'])
```

Обучение и тестовые наборы были модифицированы для различных целей. Так что будет хорошо, если разделить их еще раз:

```python
# Split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=7)
```

Вы все готовы обучать модель:

```python
# The TensorBoard extension
%tensorboard --logdir logs/
# Pass the TensorBoard callback you defined
model.fit(X_train, y_train,
         validation_data=(X_test, y_test),
         batch_size=64,
         epochs=10,
         callbacks=[tensorboard_callback],
         verbose=False)
```

Панель инструментов *TensorBoard* должна быть загружена в **Jupyter**, и вы сможете отслеживать метрику обучения и валидации.