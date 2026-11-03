모듈 = reusable Terraform configurations
providers = Terraform plugins

Terraform's configuration language is declarative
automatically calculate dependencies between resources

테라폼으로 배포하기
1. Scope - 인프라 정의
2. Author - 설정 적기
3. Initialize - 플러그인 설치
4. Plan - 미리보기한게 맞는지 확인
5. Apply - 계획 적용


terraform 설정 파일은 HCL로 작성된 일반 텍스트 파일.
파일 이름은 .tf 로 끝난대.


## 1. 인프라 생성
### 1. terraform {} 블록.

설치할 providers, 어떤 버전의 terraform을 쓸건지 등을 구성.
이 블록은 terraform.tf 파일에 넣는걸 추천.


terraform.tf
```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.92"
    }
  }

  required_version = ">= 1.2"
}
```




### 2. required_providers 블록

configuration이 사용하는 프로바이더에 대한 버전 제한을 셋팅하도록 해줌.
각 프로바이더 안에 있는 required_providers 블록에 source, version 인자를 세팅.

#### 2.1 source argument
- hostname(optional), namespace, 프로바이더 이름을 구체화함.

#### 2.2 version argument
- 안쓰면 테라폼이 최신버전을 가져오는데, 해당 구성에서 테스트 되지 않은 버전의 프로바이더를 가져오지 않도록 버전 제한을 세팅해주는걸 권장.




### 3. configuration 블록

- 새로운 테라폼 configuration을 쓸 때, provider 블록하고 다른 주요 인프라를 main.tf에 정의하는걸 권장.

#### 3.1 Providers
- provider에 의해 적용될 모든 리소스 옵션들을 구성함.


이건 aws provider를 구성하는 provider 블록
```terraform
provider "aws" {
  region = "ap-northeast-2"
}
```

- provider 블록의 라벨은 terraform 블록의 required_provider에 적힌 provider 이름과 일치해야함
- 여러 providers를 구성하거나 같은 provider에서 서로 다른 지역과 같이 다른 구성을 가지게 하기 위해 여러 provider 블록을 terraform 구성에 사용 가능
- 테라폼 provider는 리소스를 다루기 위해 cloud provider's API에 대해 인증을 제공해야함
- provider들은 여러 방식으로 인증을 제공함
- terraform의 AWS provider는 AWS CLI와 동일한 인증방식을 사용함

    - IAM 자격증명을 사용하기 위해서는 terraform AWS provider를 인증해야하고, 이를 위해서는 AWS_ACCESS_KEY_ID와 AWS_SECRET_ACCESS_KEY 환경 변수를 셋팅해야함

    ```
    export AWS_ACCESS_KEY_ID=""
    export AWS_SECRET_ACCESS_KEY=""
    ```

    - 그리고 셋팅한거 확인하기 위해서는 이렇게.
    ```
    aws configure list
    ```

#### 3.2 Data sources
- 데이터 블록을 사용해 클라우드 공급자에게 다른 리소스에 대한 정보를 쿼리가능 -> 해당 클라우드에 존재하는 다른 리소스(AMI나 인스턴스)를 "검색" 하기 위한 블록
- 필터와 일치하는 최신 AWS AMI에 대한 데이터를 가져오므로 구성에 AMI ID를 하드코딩할 필요가 없음.
- 구성을 동적으로 유지하고 오래될 수 있는 하드코딩된 값을 방지

```terraform
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }

  owners = ["099720109477"] 
}
```

- 해당 블록을 configuration 안에서 데이터 속성을 참조하도록 ID가 존재
- ID는 'data'로 시작하고 그 뒤에 블록의 타입과 이름 순서
  - ex) data.aws_ami.ubuntu = provider에게 구성된 region에서 가장 최근의 Ubuntu Noble Numbat 릴리스용 AMI를 로드


#### 3.3 Resources
- Resource 블록은 인프라의 구성 요소를 정의 -> 이거 "생성" 해줘 라는 블록

ex)AWS EC2 인스턴스를 생성하는 리소스 블록

```terraform
resource "aws_instance" "app_server" {
  ami = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name = "learn-terraform"
  }
}
```

- resource 블록의 첫번째 줄은 리소스 타입과 리소스 이름을 선언
  - ex) resource_type = aws_instance
- 리소스타입의 접두어는 provider의 이름과 일치. 나머지 문자는 provider-defined resource 타입.
- 리소스타입과 리소스 이름은 결합되어서 configuration에 있는 리소스를 위한 유니크한 리소스 주소를 형성.

- EC2 인스턴스를 위한 리소스 주소는 aws_instance.app_server
- 리소스 주소를 사용해서 configuration의 다른 부분을 사용해서 이 리소스를 참조 가능

- 리소스 블록에 있는 인자들로 리소스와 리소스의 행동을 구성
- ami : 
    data.aws_ami.ubuntu 데이터 소스의 id 속성을 언급함으로서 어떤 머신 이미지를 사용할지
- instance_type :
    AWS free tier에서 사용 가능한  t2.micro 타입을 하드코딩
- tags :
    EC2 인스턴스의 이름을 세팅. 




### 4. Format Configuration

- 읽을 수 있도록 일관된 포맷을 사용하길 권장.


```terraform
terraform fmt
```

- 커맨드는 현재 디렉토리에 있는 모든 configuration 파일들을 HashiCorp가 추천하는 스타일로 자동 재포맷함.
- 위의 커맨드 입력시 terraform은 수정할 파일들의 이름을 출력.




### 5. Initialize your workspace

- configuration을 적용하기 전에 Terraform 워크스페이스를 'terraform init' 명령어로 초기화해야함.
- 초기화 과정의 일부로, terraform 은 현재 working 디렉토리에 configuration에 정의된 providers를 설치하고 다운로드 함.

```terraform
terraform init
```

- 이럴 경우 terraform은 aws provider를 다운로드 하고 현재 working 디렉토리의 subdirectory인 숨겨진 .terraform 에 설치함.
- terraform은 또한 .terraform.lock.hcl 이라는 파일을 생성하는데, run 사이에 일관성을 유지하기 위해 workspace에서 사용되는 provider의 정확한 버전을 명시함.




### 6. Validate Configuration

- 'terraform validate' 명령어를 사용해서 구성이 구문적으로 유효하고 내부적으로 일관성이 있는지 확인하기.

```terraform
terraform validate
```

- 'validate' 명령어는 configuration에서 에러를 확인하는걸 도움.




### 7. Create Infrastructure

- terraform은 두 단계로 인프라를 바꿈.

1. 적용시킬 변경사항에 대한 실행 계획을 생성함.
- terraform이 예상대로 변경 사항을 적용하는지 확인하기 위해 이 계획을 검토할것.
- 실행 계획을 승인하면 terraform은 workspace의 providers를 사용해 해당 변화들을 적용.
- 이 workflow는 terraform이 인프라를 변화시키기 전에 예상치 못한 문제를 탐지하고 해결하도록 보장함.

2. 'terraform apply' 명령어로 configuration을 계획하고 적용할 것.
- terraform은 적용하기 전에 실행 계획을 출력하고 변경사항을 확인할 것이다.
- configuration은 'aws_instance.app_server'라는 단일 리소스를 포함하기 때문에 terraform은 EC2 인스턴스를 생성할 것이라고 알려줄 것이다.


- 출력 포멧은 Git 같은 도구에 의해 생성되는 diff 포맷과 비슷함.
- + 옆의 'resource "aws_instance" "app_server"' 는 해당 계획을 적용할때 terraform이 'aws_instance.app_server'의 ID로 리소스를 만들 예정이라는걸 의미함.
- terraform은 EC2 인스턴스에 셋팅될 속성을 보여주고 어떤 값들이 적용된 후에 알게될지 알려줌.
- terraform은 아직 어떤 인프라도 생성하지 않음.
- 만약 계획이 예상하지 못한 변경사항을 보여준다면 단계 적용을 완성하기 전에 작동을 취소할 수 있음.

- 아무튼 지금은 'yes'를 입력해서 다음 단계로 넘어가기..
- 그러면 terraform이 인프라를 생성했으니 EC2 콘솔에 가서 새로운 EC2 인스턴스를 확인해라.




### 8. Inspect state

- configuration을 적용했을 때 terraform은 'terraform.tfstate'라는 파일에 인프라에 대한 데이터를 작성.
- terraform은 상태 파일에 인프라에 대한 데이터를 저장하는데, 이 파일을 사용하여 리소스의 수명 주기 동안 리소스를 관리함
- 'terraform state list' 명령어로 terraform 워크스페이스의 상태에 있는 리소스와 데이터 소스를 나열할 수 있음

```terraform
terraform state list
```

- 데이터 소스가 실제 리소스가 아니더라도 terraform은 상태 파일을 따라갈 것이다.
- 'terraform show' 명령어를 사용해서 전체 워크스페이스의 상태를 출력하라.

```terraform
terraform show
```

- Terraform을 사용하여 워크스페이스 인프라에 대한 변경 사항을 계획하고 적용할 때, Terraform은 상태 파일의 마지막 상태, 현재 구성, 그리고 공급자가 반환한 데이터를 비교하여 실행 계획을 생성함.
- 상태 파일에는 암호나 보안 키와 같은 인프라에 대한 민감한 정보가 포함될 수 있으므로, 상태 파일을 안전하게 저장하고 Terraform을 사용하여 인프라를 관리해야 하는 사용자에게만 접근 권한을 부여해야 함
- Terraform은 기본적으로 상태 파일을 로컬에 생성
- 인프라 운영이 성숙해짐에 따라 HCP Terraform을 사용하여 상태 파일을 원격으로 저장하면 팀과의 협업이 더욱 원활해지고 상태 파일의 보안을 유지할 수 있음


---

## 2. 인프라 관리

### 1. Variables and outputs

- 입력 변수를 사용하면 Terraform 구성의 동작을 매개변수화할 수 있다.
- 출력 값을 정의하여 생성하는 리소스에 대한 데이터를 노출할 수 있다.
- 변수와 출력을 통해 Terraform 워크스페이스를 다른 자동화 도구와 통합하여 워크스페이스 인프라에 대한 데이터를 구성하고 검색하는 일관된 인터페이스를 제공할 수 있다.

#### 1.1 입력 변수 (Input variables)

variables.tf
```terraform
variable "instance_name" {
  description = "Value of the EC2 instance's Name tag."
  type        = string
  default     = "learn-terraform"
}

variable "instance_type" {
  description = "The EC2 instance's type."
  type        = string
  default     = "t3.micro"
}
```

- 이 입력 변수들은 configuration 파일을 매번 수정하지 않고 EC2 인스턴스의 이름과 타입을 업데이트 하도록 해줌
- 값을 구체화하지 않으면 두 변수 모두 terraform이 사용하도록 디폴트 값을 셋팅함
- terraform configuration을 쉽게 유지하기 위해 'variables.tf' 파일이나 'output.tf' 파일 각각에 워크스페이스의 변수와 출력 정의를 저장해두기를 추천함.
- 'main.tf' 파일에서 인스턴스 구성을 업데이트해서 변수 값을 하드코딩 하는 대신 다음 변수를 참조하도록 할 것.

```terraform
resource "aws_instance" "app_server" {
   ami           = data.aws_ami.ubuntu.id
-  instance_type = "t2.micro"
+  instance_type = var.instance_type

  tags = {
-   Name = "learn-terraform"
+   Name = var.instance_name
  }
}
```

- Terraform 변수 값은 환경 변수, 커맨드 라인 argument, 디스크에 저장된 파일 등 다양한 방법으로 설정할 수 있음.
- 커맨드 라인 변수를 사용해서 EC2 인스턴스 타입을 t2.micro 에서 t2.large로 변경했을 때 무슨 일이 일어나는지 보기 위해 적용시키는것 없이 terraform 계획을 실행시켜라



...
근데 의문인건... 저렇게 var 로 불러올 수 있으면 굳이 resource ID로 참조해야할 이유가..??
- 결론부터 말씀드리면, 변수(var)와 ID 참조(data.aws_ami.ubuntu.id 같은 것들)는 담당하는 역할과 세상에 존재하는 시점이 완전히 다릅니다.

둘의 결정적인 차이를 아주 쉽게 정리해 드릴게요!

1. 변수(var)는 언제 쓸까요? 👉 "사람이 미리 정하는 입력값"
특징: 우리가 직접 결정해서 입력하는 값입니다. ("이번에 만들 서버 타입은 t3.micro로 해줘", "이름은 learn-terraform으로 해줘" 등)
한계: 세상에 아직 존재하지 않거나, AWS 내부 사정으로 동적으로 바뀌는 값은 사람이 미리 변수로 적어둘 수가 없습니다.

2. ID 참조(data.*.id / aws_*.id)는 왜 쓸까요? 👉 "클라우드가 알려주는 동적 결과물"
여기에 ID를 참조해야 하는 이유는 크게 두 가지가 있습니다.

① ubuntu AMI ID 같은 건 매번 바뀌고 우리가 미리 알 수 없기 때문 (data 블록)
우분투 공식 이미지는 AWS가 보안 패치 등을 이유로 수시로 업데이트합니다. 그리고 서울 리전의 우분투 ID와 오리건 리전의 우분투 ID가 완전히 다릅니다.
이걸 우리가 어떻게 일일이 다 외우고 variables.tf에 기본값으로 적어둘까요? 불가능하죠!
그래서 data 블록을 써서 *"AWS야, 지금 가장 최신 우분투 AMI ID가 뭐야? 조회해 와!"*라고 물어보는 것이고, 그 결과로 나온 ID값을 data.aws_ami.ubuntu.id라는 주소로 쏙 뽑아 쓰는 것입니다.

② 서버를 만들기 전에는 그 서버의 ID를 세상에 누구도 모르기 때문 (resource 참조)
만약 EC2 인스턴스를 만들고, 그 서버에 연결할 EIP(고정 IP)나 보안 그룹을 설정한다고 해봅시다.
EC2 인스턴스는 테라폼이 apply를 실행해서 실제로 AWS에 만들어야 비로소 고유한 ID(예: i-0123456789abcdef0)가 발급됩니다.
만들기 전에는 그 ID가 세상에 존재하지 않으니 변수로 미리 쓸 수가 없겠죠? 그래서 테라폼에게 *"앞서 만든 그 EC2 인스턴스의 ID를 가져다 써라"*라고 명시적으로 연결해 주는 것입니다.

🎯 한 줄 요약
변수(var): 사람이 직접 입력하는 설정값 (t3.micro, 이름표 등)
ID 참조 (.id): AWS가 동적으로 조회해 주거나 생성해 준 결과물 (AMI ID, 생성된 서버 고유 번호 등)
즉, 변수는 "어떤 스펙으로 만들까?"를 정할 때 쓰고, ID 참조는 "방금 조회했거나 만들어진 녀석의 고유 주소(ID)를 가져다 쓸게"라고 할 때 쓰는 전혀 다른 용도인 것입니다!


```terraform
terraform plan -var instance_type=t2.large
```

- 해당 계획을 적용하면 terraform은 EC2 인스턴스를 새로운 타이븡로 업데이트 할 것이다.
- terraform은 이 변화를 이행하기 위해 인스턴스를 교체해야할 필요가 있기 때문에 AWS provider는 또한 IP 주소와 hostname도 바뀔 것이라고 가리키지만 적용 후에 알 수 있다고 표시하는데 왜냐면 AWS 는 인스턴스를 다시 만들기 위해 변경사항을 적용하기 전까지는 새로운 값을 할당하지 않기 때문이다.


#### 1.2 출력 변수 (Output variables)
- 출력 변수는 terraform configuration의 속성에 접근하도록 하고 다른 자동화 도구나 workflows로 이 값들을 소비하도록 한다.
- 'outputs.tf' 라는 파일을 만들고 다음을 설정.

```terraform
output "instance_hostname" {
  description = "Private DNS name of the EC2 instance."
  value       = aws_instance.app_server.private_dns
}
```

- 해당 출력값은 terraform workspace에서의 EC2 인스턴스의 hostname을 출력할 것이다.

- configuration을 적용해라.

```terraform
terraform apply
```

- 생성한 두 변수의 기본값이 기존의 하드코딩된 값과 동일하므로 Terraform은 추가한 출력 값만 변경된 것으로 감지한다.
- 출력 값을 작업 공간에 추가하려면 확인 메시지에 '예'라고 응답하십시오.

- terraform은 계획을 작동시키거나 적용할 때 출력 값을 출력할 것이고 이걸 workspace의 상태 파일에 저장할 것이다.

```terraform
terraform output
```




### 2. Modules

- 모듈은 재사용가능한 configuration 세트.
- 모듈을 사용하면 여러 리소스와 데이터 소스를 포함하는 복잡한 인프라 배포를 일관되게 관리할 수 있음.
- provider처럼, Terraform 레지스트리에서 모듈을 가져올 수 있음.
- 자체 모듈을 생성하여 조직 내에서 공유할 수도 있음.


#### 2.1 Module 블록
- VPC와 EC2 인스턴스와 연결된 네트워킹 리소스를 만들기 위해 'main.tf' 파일에 모듈 블록을 추가해라.

```terraform
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.19.0"

  name = "example-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24"]

  enable_dns_hostnames    = true
}
```

- terraform이 configuration에 있는 의존성들을 자동으로 해결하기 때문에 어떤 순서로든 configuration을 구성할 수 있음.
- 이 configuration은 'example-vpc'라는 이름의 VPC를 두 개의 퍼블릭 서브넷과 두 개의 프라이빗 서브넷으로 정의함.
- terraform 레지스트리에 있는 AWS VPC 모듈 페이지를 검토하면, 서브넷, 보안 그룹, 탄력적 IP 주소 및 NAT 게이트웨이를 포함한 네트워킹 리소스를 생성하는 데 모듈을 사용하는 방법에 대한 설명서와 예제가 포함되어 있음.
- 아래처럼 리소스 블록을 업데이트해서 EC2 인스턴스를 새로운 VPC로 옮겨라.

```terraform
resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  vpc_security_group_ids = [module.vpc.default_security_group_id]
  subnet_id              = module.vpc.private_subnets[0]

  tags = {
    Name = var.instance_name
  }
}
```

- 이 변경사항은 EC2 인스턴스가 새로운 VPC의 퍼블릿 서브넷에 있도록 하고 디폴트 security group을 사용하게 할 것이다.
- 구성에 새 모듈을 추가할 때마다 작업 공간을 다시 초기화하여 해당 모듈을 설치해야 함.
- 'terraform init'을 입력해서 VPC 모듈을 설치해라.

- 기존 워크스페이스를 초기화하면 Terraform은 새로운 프로바이더와 모듈을 감지하고 설치함. 
- Terraform은 워크스페이스 디렉터리의 .terraform.lock.hcl 파일에서 구성에 사용된 프로바이더의 현재 버전을 추적.




### 3. Plan and apply changes

- 업데이트된 구성을 적용하여 새 VPC를 생성하고 EC2 인스턴스를 해당 VPC로 이동합니다. 
- AWS는 기존 EC2 인스턴스를 새 VPC로 이동하는 것을 지원하지 않으므로 Terraform은 기존 인스턴스를 제자리에서 업데이트하는 대신 교체할 계획을 세웁니다. 
- 확인 메시지가 나타나면 '예'를 선택하여 Terraform의 VPC 추가 및 EC2 인스턴스 교체 계획을 승인하십시오.

```terraform
terraform apply
```

- Terraform은 추가한 vpc 모듈에 정의된 VPC 및 관련 리소스를 생성함.
- Terraform은 리소스 종속성을 기반으로 필요한 작업을 순서대로 수행.
-  먼저 기존 EC2 인스턴스를 삭제한 다음 VPC와 대부분의 관련 리소스를 생성하고 마지막으로 새 EC2 인스턴스를 생성.
- Terraform은 실행 계획을 생성할 때 종속성 그래프를 구성하여 올바른 작업 순서를 결정합니다. 
- 계획을 적용하면 Terraform은 종속성 순서대로 리소스를 생성, 업데이트 및 삭제하며, 가능한 경우 병렬로 처리함.
- 'terraform state list' 명령을 사용하여 워크스페이스의 리소스 목록을 출력할 수 있음.

```terraform
terraform state list
```

- VPC 모듈 내에 구성된 리소스는 `module.vpc`로 시작한다는 점에 유의.
- 리소스 및 데이터 소스 주소와 마찬가지로 모듈 주소도 구성 내에서 고유해야 함.
- 동일한 모듈 블록에 고유한 이름을 지정하고 동일한 `source` 및 `version` 인수를 지정하면 동일한 모듈을 여러 번 사용할 수 있음.


---

## 3. Destroy Infrastructure

- 워크스페이스에서 관리하는 인프라가 더 이상 필요하지 않으면 Terraform을 사용하여 삭제하세요. 
- Terraform은 구성에서 리소스를 제거하고 변경 사항을 적용하면 해당 리소스를 삭제합니다. 
- 또한 `terraform destroy` 명령을 사용하여 구성에서 관리하는 모든 리소스를 한 번에 삭제할 수도 있습니다.

### 1. Remove a resource
- EC2 인스턴스를 configuration에서 제거하기 위해 `main.tf` 파일을 열어 `aws_instance.app_server`리소스 블록을 주석처리 할 것.
- `outputs.tf` 파일의 `instance_hostname` 출력 값은 `main.tf`에서 제거한 EC2 인스턴스를 참조하므로 해당 출력 값을 주석 처리할 것. 그렇지 않으면 구성이 유효하지 않게 됨.
- `terraform apply` 를 실행시켜 해당 변경을 적용할 것
- Terraform이 인스턴스를 삭제하고 출력 값을 제거하는 계획을 승인하려면 확인 메시지에 '예'라고 응답하십시오.




### 2. Destroy workspace

- 워크스페이스에서 개별 리소스를 제거하는 것은 인프라 관리 워크플로의 일반적인 부분입니다. 
- 워크스페이스에서 더 이상 필요하지 않은 인프라가 있으면 Terraform을 사용하여 해당 리소스를 삭제할 것. 
- 예를 들어 애플리케이션 환경을 서비스에서 제거하거나, 빌드 또는 테스트 시스템과 같이 수명이 짧은 환경을 관리하기 위해 Terraform을 사용할 수 있습니다.

```terraform
terraform destroy
```

- `terraform destroy` 를 실행시켜 workspace를 제거하십시오.

---

## 4. Collaborate using HCP Terraform

- HCP Terraform은 조직내에서 인프라 프로젝트에 대해 협업하도록 도움
- 안전한 원격 실행 환경을 제공하여 인프라 변경 사항을 공동으로 관리할 수 있도록 함
- API 토큰 및 액세스 키를 포함한 Terraform 상태 및 변수를 안전하게 저장
- Terraform CLI는 HCP Terraform과 통합되어 로컬 머신에서 원격 실행을 트리거 가능

### 1. 리소스 생성

```terraform
terraform apply
```




### 2. Sign up for HCP Terraform
- 계정 생성, organization 생성...




### 3. Log into HCP Terraform

```terraform
terraform login
```

- api 토큰 만들고... 로그인 하고...




### 4. Connect workspace to HCP Terraform

- 워크스페이스를 HCP Terraform에 연결하려면 구성을 업데이트하고 커맨드라인에서 워크스페이스를 다시 초기화해야 함
- HCP Terraform 워크스페이스는 로컬 Terraform 워크스페이스와 유사하게 작동하지만, Terraform 작업을 실행하고 상태를 HCP Terraform 내에 저장
- HCP Terraform은 워크스페이스를 프로젝트로 그룹화하여 액세스 및 설정을 보다 쉽게 ​​관리할 수 있도록 지원

#### 4.1 configuration 업데이트
- 로컬 workspace를 HCP Terraform과 연결하기 위해 `terraform.tf` 파일에 cloud 블록을 추가

terraform.tf
```terraform
terraform {
  cloud {
    organization = "your-organization-name"

    workspaces {
      project = "Learn Terraform"
      name = "learn-terraform-aws-get-started"
    }
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.92"
    }
  }

  required_version = ">= 1.2.0"
}

```

#### 4.2 HCP Terraform 으로 migration
- HCP Terraform 통합 구성이 완료되었으니 `terraform init` 명령 실행으로 구성을 다시 초기화하고 상태를 HCP Terraform으로 복사할 것
- 마이그레이션을 확인하려면 확인 메시지에 `yes`라고 응답하십시오.

```terraform
terraform init
```

- 클라우드 블록을 구성에 추가한 후 `terraform init` 명령을 실행할 경우, Terraform은 프로젝트와 워크스페이스가 존재하지 않으면 새 프로젝트와 워크스페이스를 생성하고 이미 존재하는 경우 기존 프로젝트와 워크스페이스를 사용

#### 4.3 Configure AWS credentials
- `terraform init`했을때 HCP Terraform organization에 `Learn Terraform` 프로젝트와 `learn-terraform-aws-get-started` 워크스페이스가 생성
- HCP Terraform이 AWS 공급자와 인증하고 리소스를 프로비저닝하려면 AWS 자격 증명으로 워크스페이스를 구성해야함
- HCP Terraform에서 learn-terraform-aws-get-started 워크스페이스로 이동한 다음 워크스페이스의 변수 페이지로 이동
- 워크스페이스 변수에서 AWS_ACCESS_KEY_ID와 AWS_SECRET_ACCESS_KEY를 환경 변수로 추가하고, 반드시 민감 변수로 표시할 것!

#### 4.4 Apply configuration
- HCP Terraform을 사용하여 구성을 업데이트하고 적용
- main.tf 파일에서 EC2 인스턴스에 대한 구성의 주석 처리를 해제, output.tf 에서의 주석도 해제...
- 그리고 HCP Terraform 에서 실행시키기 위해 `terraform apply` 실행.
- Terraform은 HCP Terraform에서 원격으로 계획 및 적용 작업을 실행하고 출력을 터미널 창으로 스트리밍
- 확인 메시지가 나타나면 '예'를 선택하여 HCP Terraform이 인프라를 프로비저닝하도록 지시할 것

```terraform
terraform apply
```

- Terraform은 워크스페이스 상태를 HCP Terraform에 원격으로 저장
- Terraform이 출력하는 URL을 방문하면 HCP Terraform에서 실행 결과를 확인 가능
- CLI 기반 워크플로를 사용하면 로컬 머신에서 Terraform 명령을 실행할 수 있으며, HCP Terraform이 해당 명령을 실행하고 출력을 CLI로 스트리밍
- HCP Terraform에서 계획 및 적용 실행 결과를 검토하고 워크스페이스를 구성 가능

---

### 5. Destroy workspace

- 이 튜토리얼에서 프로비저닝한 리소스를 삭제하려면 `terraform destroy` 명령을 실행
- 확인 메시지가 나타나면 '예'를 선택하여 HCP Terraform이 워크스페이스에서 관리하는 모든 리소스를 삭제하도록 지시

```terraform 
terraform destroy
```

- HCP Terraform에서 learn-terraform-aws-get-started 워크스페이스의 설정 > 삭제 탭으로 이동하여 워크스페이스를 삭제
- HCP Terraform에서 삭제 버튼을 클릭한 다음, 안내에 따라 워크스페이스를 삭제

- 프로젝트 설정 페이지로 이동하여 삭제 버튼을 클릭하고 안내에 따라 진행하면 프로젝트를 삭제할 수 있음