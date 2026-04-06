# Azure ML 모델 다운로드

Azure ML에 등록된 모델을 **Registry** 또는 **Workspace**에서 다운로드하는 예제입니다.

## 프로젝트 구조

```
download-model/
├── from-registry/    # Azure ML Registry에서 모델 다운로드
└── from-workspace/   # Azure ML Workspace에서 모델 다운로드
```

## 사전 요구사항

- Python 3.10+
- Azure 구독 및 인증 설정 (`az login`)

```bash
pip install azure-ai-ml azure-identity mlflow python-dotenv
```

## 사용 방법

1. 각 폴더의 `.env` 파일에 환경변수를 설정합니다.
2. Jupyter Notebook을 실행합니다.

### Registry에서 다운로드

```
REGISTRY_NAME=<your-registry-name>
MODEL_NAME=<your-model-name>
MODEL_VERSION=<your-model-version>
```

### Workspace에서 다운로드

```
SUBSCRIPTION_ID=<your-subscription-id>
RESOURCE_GROUP=<your-resource-group>
WORKSPACE_NAME=<your-workspace-name>
MODEL_NAME=<your-model-name>
MODEL_VERSION=<your-model-version>
```

## Registry vs Workspace 비교

| 항목 | Registry | Workspace |
|------|----------|-----------|
| 범위 | 구독/테넌트 간 공유 가능 | 특정 Workspace 내 한정 |
| MLClient 연결 | `registry_name` | `subscription_id` + `resource_group_name` + `workspace_name` |
| 용도 | 조직 전체 모델 공유·배포 | 개별 팀/프로젝트 실험·관리 |
| 필요 환경변수 | `REGISTRY_NAME` | `SUBSCRIPTION_ID`, `RESOURCE_GROUP`, `WORKSPACE_NAME` |

## 필요 권한 (RBAC)

### Registry에서 모델 다운로드

Registry는 자체 관리 Storage를 사용하므로, 별도의 Storage 역할이 필요 없습니다.

| 구분 | 역할 이름 | 할당 범위 | 이유 |
|------|----------|----------|------|
| 필수 | **AzureML Registry User** | Registry | Registry 조회 + assets 읽기/다운로드 (`registries/read`, `registries/assets/*`) |

상위 역할인 Contributor, Owner도 사용 가능합니다.

### Workspace에서 모델 다운로드

`ml_client.models.download()` 호출 시 필요한 권한은 Workspace **Datastore의 인증 방식**에 따라 다릅니다.

#### Credential-based Datastore (기본값)

Workspace 생성 시 기본 datastore(`workspaceblobstore`)는 **account key** 기반으로 생성됩니다.
이 경우 datastore에 저장된 키를 통해 Storage에 접근하므로, Storage 수준의 별도 역할이 필요 없습니다.

| 구분 | 역할 이름 | 할당 범위 | 이유 |
|------|----------|----------|------|
| 필수 | **AzureML Data Scientist** | Workspace | 모델 메타데이터 조회 + datastore 저장된 credential로 파일 다운로드 (`workspaces/*/read`, `workspaces/*/action` 포함) |

> `AzureML Data Scientist` 역할에는 `datastores/listsecrets/action` 권한이 포함되어 있어, datastore에 저장된 Storage 키를 사용하여 모델 파일을 다운로드할 수 있습니다.

#### Identity-based Datastore

Datastore가 **credential 없이(identity-based)** 구성된 경우, 사용자의 Entra ID로 직접 Storage에 접근합니다.
이 경우 Storage 수준의 역할이 **추가로** 필요합니다.

| 구분 | 역할 이름 | 할당 범위 | 이유 |
|------|----------|----------|------|
| 필수 1 | **AzureML Data Scientist** | Workspace | 모델 메타데이터(이름, 버전, 저장 경로) 조회 |
| 필수 2 | **Storage Blob Data Reader** | 연결된 Storage Account | 실제 모델 파일(.pkl, .bin 등)을 Blob Storage에서 다운로드 |

#### 역할별 비교

| 역할 | 모델 메타데이터 조회 | Credential-based 다운로드 | Identity-based 다운로드 |
|------|:---:|:---:|:---:|
| Reader | ✅ | ❌ (`listsecrets` 없음) | ❌ (Storage 권한 없음) |
| AzureML Data Scientist | ✅ | ✅ | ❌ (Storage 권한 별도 필요) |
| AzureML Data Scientist + Storage Blob Data Reader | ✅ | ✅ | ✅ |
| Contributor | ✅ | ✅ | ❌ (Storage 권한 별도 필요) |

## 참고 문서

- [Manage access to Azure ML workspaces](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-assign-roles?view=azureml-api-2)
- [Manage Azure ML registries](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-manage-registries?view=azureml-api-2)
- [Data administration](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-administrate-data-authentication?view=azureml-api-2)
- [Azure built-in roles for AI + ML](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/ai-machine-learning)
