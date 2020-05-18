---
title: KFServing
description: "Kubeflow Serving Component"
categories: [Kubeflow, ML Serving]
---

# [KFServing](https://www.kubeflow.org/docs/components/serving/kfserving/)

- Kubernetes [Custom Resource Definition(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 으로 제공되는 ML Serving 아키텍처

![KFServing](https://www.kubeflow.org/docs/components/serving/kfserving.png)

## InferenceService

###### ![Data Plane](https://github.com/kubeflow/kfserving/raw/master/docs/diagrams/dataplane.jpg)

- KFServing의 배포/서비스 단위 (CRD)
- api는 [Tensorflow V1 HTTP API](https://www.tensorflow.org/tfx/serving/api_rest#predict_api)를 따름
- Advanced features such as Ensembling, A/B testing, and Multi-Arm-Bandits should compose InferenceServices together 
  - 한편 seldon은...[(p28)](https://www.slideshare.net/seldon_io/seldon-deploying-models-at-scale)

## InferenceService 구성요소

### Predictor

- 필수
- predict를 수행하는 model만 있음 

### Transformer

- prediction 또는 explanation의 실행 전/후에 수행될 로직이 탑재

## InferenceService 유형

### [Out-of-the-box Predictor](https://github.com/kubeflow/kfserving/tree/master/docs/samples#deploy-kfserving-inferenceservice-with-out-of-the-box-predictor)

#### tensorflow

```yaml
  
apiVersion: "serving.kubeflow.org/v1alpha2"
kind: "InferenceService"
metadata:
  name: "flowers-sample"
spec:
  default:
    predictor:
      tensorflow:
        storageUri: "gs://kfserving-samples/models/tensorflow/flowers"
```

#### pytorch

```yaml
apiVersion: "serving.kubeflow.org/v1alpha2"
kind: "InferenceService"
metadata:
  name: "pytorch-cifar10"
spec:
  default:
    predictor:
      pytorch:
        storageUri: "gs://kfserving-samples/models/pytorch/cifar10/"
        modelClassName: "Net"
```

### Custom Predictor
[Flask Hello World 예제](https://github.com/kubeflow/kfserving/tree/master/docs/samples/custom/hello-world)

```yaml
apiVersion: serving.kubeflow.org/v1alpha2
kind: InferenceService
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: custom-sample
spec:
  default:
    predictor:
      custom:
        container:
          image: sds.redii.net/sample/custom-sample
          env:
            - name: GREETING_TARGET
              value: "Python KFServing Sample"
```

### [Transformer](https://github.com/kubeflow/kfserving/tree/master/docs/samples/transformer/image_transformer)

```yaml
apiVersion: serving.kubeflow.org/v1alpha2
kind: InferenceService
metadata:
  name: transformer-cifar10
spec:
  default:
    transformer:
      custom:
        container:
          image: gcr.io/kubeflow-ci/kfserving/image-transformer:latest
          name: user-container  
    predictor:
      pytorch:
        modelClassName: Net
        storageUri: gs://kfserving-samples/models/pytorch/cifar10
```

```python
def image_transform(instance):
    ...생략...
    return res.tolist()
  
class ImageTransformer(kfserving.KFModel):
    def __init__(self, name: str, predictor_host: str):
        super().__init__(name)
        self.predictor_host = predictor_host

    def preprocess(self, inputs: Dict) -> Dict:
        return {'instances': [image_transform(instance) for instance in inputs['instances']]}

    def postprocess(self, inputs: List) -> List:
        return inputs
      
if __name__ == "__main__":
    transformer = ImageTransformer(args.model_name, predictor_host=args.predictor_host)
    kfserver = kfserving.KFServer()
    kfserver.start(models=[transformer])      
```

## 배포

- [Pipelines](https://github.com/kubeflow/kfserving/blob/master/docs/samples/pipelines/sample-tf-pipeline.py)

- kubectl

  ```
  kubectl apply -f tensorflow.yaml 
  ```

  

## 참고: TensorFlow SavedModel

https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/saved_model/README.md

```python
export_dir = ...
...
builder = tf.saved_model.builder.SavedModelBuilder(export_dir)
with tf.Session(graph=tf.Graph()) as sess:
  ...
  builder.add_meta_graph_and_variables(sess,
                                       [tf.saved_model.tag_constants.TRAINING],
                                       signature_def_map=foo_signatures,
                                       assets_collection=foo_assets)
...
with tf.Session(graph=tf.Graph()) as sess:
  ...
  builder.add_meta_graph(["bar-tag", "baz-tag"])
...
builder.save()
```

- [SignatureDef](https://github.com/tensorflow/serving/blob/master/tensorflow_serving/g3doc/signature_defs.md)로 다음 항목을 정의
  - method_name
  - inputs: 입력 tensor의 이름, dtype, shape을 정의
  - outpus: 출력  tensor의 이름, dtype, shape을 정의
- 다음과 같은 구조로 모델이 저장됨


```
assets/
assets.extra/
variables/
    variables.data-?????-of-?????
    variables.index
saved_model.pb
```

