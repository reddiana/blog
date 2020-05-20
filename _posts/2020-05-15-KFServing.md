---
title: KFServing
description: "Kubeflow Serving Component"
categories: [Kubeflow, ML Serving]
---

[TOC]

# [KFServing](https://www.kubeflow.org/docs/components/serving/kfserving/)

- Kubernetes [Custom Resource Definition(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 으로 제공되는 ML Serving 아키텍처
- serverless 
  - 내부적으로 Knative 사용

![KFServing](https://www.kubeflow.org/docs/components/serving/kfserving.png) 

## 1. InferenceService

![Data Plane](https://github.com/kubeflow/kfserving/raw/master/docs/diagrams/dataplane.jpg) 

- KFServing의 배포/서비스 단위 (CRD)
- api는 [Tensorflow V1 HTTP API](https://www.tensorflow.org/tfx/serving/api_rest#predict_api)를 따름 ([Out-of-the-box](https://m.blog.naver.com/yangseungjae/220708695668)의 경우)
- Ensembling, A/B testing, Multi-Arm-Bandits 등은 InferenceService를 조합하여 구현해야함 (단일 InferenceService로는 불가능)
  - Seldon Core와 비교됨 (참고2)

## 2. InferenceService 구성요소

### 2.1 Predictor

- 필수
- REST API path의 postfix가 `:predict`
- model(예: Tensorflow SavedModel)의 serving만 있음
  - storage에 저장된 model을 load하여 서비스
    - 지원하는 storage provider
      - Google Cloud Storage `gs://`
      - S3 Compatible Object Storage `s3://`
      - Azure Blob Storage `https://`
      - Local filesystem either `file://`
      - Persistent Volume Claim (PVC) `pvc://`
  - model은 프로비저닝되어 제공되는 이미지(framework)에 따름
    - 제공되는 이미지 프로비저닝 
      - 

### 2.2 Transformer

- 필수아님
- prediction 또는 explanation의 실행 전/후에 수행될 로직이 탑재

### 2.3 Explainer

- 필수아님
- REST API path의 postfix가 `:explain`
- 제공되는 이미지 프로비저닝
  - [Seldon Alibi](https://www.seldon.io/tech/products/alibi/)

## 3. InferenceService 배포

- 방법1: kubectl

  ```bash
  kubectl apply -f xxxxx.yaml 
  ```

- 방법2: Kubeflow Pipelines ([Sample](https://github.com/kubeflow/kfserving/blob/master/docs/samples/pipelines/sample-tf-pipeline.py))

- 방법3: [KFServing Python SDK](https://github.com/kubeflow/kfserving/blob/master/python/kfserving) ([Sample](https://github.com/kubeflow/kfserving/blob/master/docs/samples/client/kfserving_sdk_sample.ipynb))

## 4. InferenceService 유형

- 구성요소에 따른 구분
  - Predictor
  - Transformer (Custom만 있음)
  - Explainer
- 이미지 프로비저닝 여부에 따른 구분
  - [Out-of-the-box](https://m.blog.naver.com/yangseungjae/220708695668) <- 프로비저닝된 이미지 사용
  - Custom

### 4.1 [Out-of-the-box Predictor](https://github.com/kubeflow/kfserving/tree/master/docs/samples#deploy-kfserving-inferenceservice-with-out-of-the-box-predictor)

- 프로비저닝 이미지를 사용
  - 프로비저닝할 프레임워크(예: tensorflow, pytorch) 명시 필요 (이미지에 대한 기술 X) 
- 모델이 저장된 위치 필요 (storageUri)

##### 4.1.1 tensorflow 예제 yaml

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

##### 4.1.2 pytorch 예제 yaml

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

### 4.2 [Custom Predictor](https://github.com/kubeflow/kfserving/tree/master/docs/samples/custom)

- 프로비저닝 이미지를 사용하지 않고 Custom 이미지를 사용

##### 4.2.1 [Flask Hello World](https://github.com/kubeflow/kfserving/tree/master/docs/samples/custom/hello-world) 예제 yaml

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

##### 4.2.2 [kfserving-custom-model](https://github.com/kubeflow/kfserving/tree/master/docs/samples/custom/kfserving-custom-model) 예제

- kfserving.KFModel을 상속받고 load()와 predict()를 구현함

```python
import kfserving
from typing import List, Dict

class KFServingSampleModel(kfserving.KFModel):
    def __init__(self, name: str):
        super().__init__(name)
        self.name = name
        self.ready = False

    def load(self):
        # model load 코드 생략
        self.model = model
        self.ready = True

    def predict(self, request: Dict) -> Dict:
        inputs = request["instances"]
        # predict 코드 생략
        return {"predictions": results}

if __name__ == "__main__":
    model = KFServingSampleModel("kfserving-custom-model")
    model.load()
    kfserving.KFServer(workers=1).start([model])
```

### 4.3 [Transformer](https://github.com/kubeflow/kfserving/tree/master/docs/samples/transformer/image_transformer)

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

- kfserving.KFModel을 상속받고 preprocess()와 postprocess()를 구현함

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

## 5.  [KFServing Python SDK](https://github.com/kubeflow/kfserving/blob/master/python/kfserving)

### 5.1 Model / Server

- "4.2 [Custom Predictor](https://github.com/kubeflow/kfserving/tree/master/docs/samples/custom) / 4.2.2 [kfserving-custom-model](https://github.com/kubeflow/kfserving/tree/master/docs/samples/custom/kfserving-custom-model) 예제" 및 "4.3 [Transformer](https://github.com/kubeflow/kfserving/tree/master/docs/samples/transformer/image_transformer)" 참조

### 5.2 Client

- "방법3: KFServing Python SDK ([Sample](https://github.com/kubeflow/kfserving/blob/master/docs/samples/client/kfserving_sdk_sample.ipynb))" 참조

## 참고1: TensorFlow SavedModel

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

## 참고2: [Seldon Core](https://docs.seldon.io/projects/seldon-core/en/stable/) Inference Graph(Pipeline)

![graph](2020-05-15-KFServing.assets/seldon-core-high-level.jpg) 

- https://docs.seldon.io/projects/seldon-core/en/stable/graph/inference-graph.html
- https://www.slideshare.net/seldon_io/seldon-deploying-models-at-scale (p28)

