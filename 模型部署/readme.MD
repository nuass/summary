Tensorflow 模型服务主要支持gRPC API，同时也支持RESTful API。以下介绍gRPC和RESTful接口两种实现方式。

###gRPC模型的部署预测
对于模型部署代码一般主要分为两部分，模型的训练、保存文件和客户端，tensorflow部署所用模型多为pb格式（[ckpt格式](http://192.168.50.30/index.php?s=/page/1088 "ckpt格式")更多地是本地预测）。
以[CNN serving ](http://192.168.50.14:8081/xpcloud/deeplearn/tree/42eead9affb8ca3a8d30ee0dc3f45298f1f38bd3/serving/cnn_serving_mnist "样本案例")为例做介绍。

- 训练、pb保存模型、导出

mnist_saved_model.py训练部分与其他训练一样，以下主要是保存

```

#...............训练过程省略............
#..............以下pb导出过程.............

    builder = tf.saved_model.builder.SavedModelBuilder(export_path)

    # Build the signature_def_map.
    classification_inputs = tf.saved_model.utils.build_tensor_info(
      serialized_tf_example)
    classification_outputs_classes = tf.saved_model.utils.build_tensor_info(
      prediction_classes)
    classification_outputs_scores = tf.saved_model.utils.build_tensor_info(values)

    classification_signature = (
      tf.saved_model.signature_def_utils.build_signature_def(
          inputs={
              tf.saved_model.signature_constants.CLASSIFY_INPUTS:
                  classification_inputs},
          outputs={
              tf.saved_model.signature_constants.CLASSIFY_OUTPUT_CLASSES:
                  classification_outputs_classes,
              tf.saved_model.signature_constants.CLASSIFY_OUTPUT_SCORES:
                  classification_outputs_scores},
          method_name=tf.saved_model.signature_constants.CLASSIFY_METHOD_NAME))

    tensor_info_x = tf.saved_model.utils.build_tensor_info(x)
    tensor_info_y = tf.saved_model.utils.build_tensor_info(y)

    prediction_signature = (
      tf.saved_model.signature_def_utils.build_signature_def(
          inputs={'images': tensor_info_x},
          outputs={'scores': tensor_info_y},
          method_name=tf.saved_model.signature_constants.PREDICT_METHOD_NAME))

    legacy_init_op = tf.group(tf.tables_initializer(), name='legacy_init_op')
    builder.add_meta_graph_and_variables(
      sess, [tf.saved_model.tag_constants.SERVING],
      signature_def_map={
          'predict_images':
              prediction_signature,
          tf.saved_model.signature_constants.DEFAULT_SERVING_SIGNATURE_DEF_KEY:
              classification_signature,
      },
      legacy_init_op=legacy_init_op)

    builder.save()

```


运行方式

```
python mnist_saved_model.py --training_iteration=100 
      --model_version=1 --model_dir=./model

```

得到模型如下.
```python
root@myyunbj2:~/root/project/cnn_serving_mnist/model/1# ls
saved_model.pb  variables

```
注意tensorflow serving 导出的格式模型合适为.pb


- 部署运行（开启服务）
tensorflow模型服务库通过tensorflow_model_server命令运行

**tensorflow_model_server 主要参数**

|  参数名 |类型   | 说明  |
| ------------ | ------------ | ------------ |
| - -port=8500  |  int32 | 监听 gRPC API 端口  |
| - -rest_api_port=0    |  int32 | 监听 HTTP/REST API 端口  |
|  - -model_name="default" |  string | 模型名称  |
|   - -model_base_path=""| string  | 模型路径（绝对路径、且到数字文件的上一级） |


    
```python
tensorflow_model_server --port=8888 --model_name=mnist --model_base_path=/root/project/cnn_serving_mnist/model

```




- 部署成功部分log
2018-07-30 19:07:51.578957: I tensorflow_serving/model_servers/main.cc:153] Building single TensorFlow model file config:  model_name: mnist model_base_path: /root/project/serving/data/mnist_model
2018-07-30 19:07:51.579123: I tensorflow_serving/model_servers/server_core.cc:459] Adding/updating models.
2018-07-30 19:07:51.579150: I tensorflow_serving/model_servers/server_core.cc:514]  (Re-)adding model: mnist
...
...
tensorflow_serving/servables/tensorflow/saved_model_warmup.cc:83] No warmup data file found at /root/project/serving/data/mnist_model/1/assets.extra/tf_serving_warmup_requests
2018-07-30 19:07:51.699598: I tensorflow_serving/core/loader_harness.cc:86] Successfully loaded servable version {name: mnist version: 1}
2018-07-30 19:07:51.701185: I tensorflow_serving/model_servers/main.cc:323] Running ModelServer at 0.0.0.0:8888 ...


- mnist_client.py 预测

```
  test_data_set = mnist_input_data.read_data_sets(work_dir).test
  host, port = hostport.split(':')
  channel = implementations.insecure_channel(host, int(port))
  stub = prediction_service_pb2.beta_create_PredictionService_stub(channel)
  result_counter = _ResultCounter(num_tests, concurrency)
  for _ in range(num_tests):
    request = predict_pb2.PredictRequest()
    request.model_spec.name = 'mnist'
    request.model_spec.signature_name = 'predict_images'
    image, label = test_data_set.next_batch(1)
    request.inputs['images'].CopyFrom(
        tf.contrib.util.make_tensor_proto(image[0], shape=[1, image[0].size]))
    result_counter.throttle()
    result_future = stub.Predict.future(request, 5.0)  # 5 seconds
    result_future.add_done_callback(
        _create_rpc_callback(label[0], result_counter))
  return result_counter.get_error_rate()

```



在部署成功后进行预测，注意port需要和部署的一致。
```
python ./mnist_client.py --num_tests=1000 --server=localhost:8888
```
预测结果
![](http://192.168.50.30/Public/Uploads/2018-07-30/5b5ef942e11f4.PNG)





##Restful API

tensorflow[官方示例](https://www.tensorflow.org/serving/api_rest "官方示例")对RESTful有介绍，以下是部署预测过程。

训练模型，可以直接使用已训练好的[下载地址](https://github.com/tensorflow/serving/tree/master/tensorflow_serving/servables/tensorflow/testdata/saved_model_half_plus_three/00000123 "下载地址")。

- 模型

```
xxx@xxxx:~/saved_model_half_plus_three/00000123$ ls
assets  saved_model.pb  variables


```
- 启动tensoeflow server model

```
tensorflow_model_server --rest_api_port=8501 \
   --model_name=half_plus_three \
   --model_base_path=/home/xxxx/saved_model_half_plus_three/
```
- REST API calls

```
curl http://localhost:8501/v1/models/half_plus_three
```

- 预测及结果

```
#post请求
curl -d '{"instances": [1.0,2.0,5.0]}' -X POST http://localhost:8501/v1/models/half_plus_three:predict
#返回结果
{
    "results": [3.5, 4.0]
}
```




####安装
pip方式：(其他安装见[官方文档](https://www.tensorflow.org/serving/setup "官方安装文档"))


```
pip install tensorflow-serving-api
#需要curl
echo "deb [arch=amd64] http://storage.googleapis.com/tensorflow-serving-apt stable tensorflow-model-server tensorflow-model-server-universal" | sudo tee /etc/apt/sources.list.d/tensorflow-serving.list


curl https://storage.googleapis.com/tensorflow-serving-apt/tensorflow-serving.release.pub.gpg | sudo apt-key add -
#

sudo apt-get update && sudo apt-get install tensorflow-model-server
```



####参考：
docker 发布服务https://www.tensorflow.org/serving/serving_inception
https://github.com/tensorflow/serving/blob/master/tensorflow_serving/g3doc/serving_basic.md
https://github.com/tensorflow/tensor2tensor/issues/687
https://github.com/tensorflow/serving/issues/549
https://docs.bitnami.com/bch/how-to/enable-nvidia-gpu-tensorflow-serving/
AWS grpc、 AWS Lambda
https://docs.aws.amazon.com/dlami/latest/devguide/model-serving.html
阿里云 grpg /CPU，GPU可选择，port可选择
https://helpcdn.aliyun.com/document_detail/52694.html


