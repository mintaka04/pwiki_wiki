# 리소스에 대해


- terraform은 가상 네트워크나 compute 인스턴스, 또는 DNS 레코드와 같은 고레벨의 요소 같은 인프라를 다루기 위해 리소스 블록을 사용
- 리소스 블록은 terraform configuration에 있는 하나 이상의 인프라 objects를 나타냄.


- 대부분 terraform providers는 특정한 인프라 타입을 관리하기 위한 적절한 APIs에 매핑되는 여러 리소스를 가짐

generic|AWS Provider Resource|AWS 인프라
-|-|-
Resource A|aws_instance|EC2 Instance
Resource B|aws_security_group|Security Group


- 튜토리얼에서는 PHP 웹 애플리케이션을 실행시키기 위한 EC2 인스턴스를 만들 예정.
- 그런 다음 Terraform 레지스트리의 문서를 참조하여 애플리케이션을 공개적으로 액세스할 수 있도록 보안 그룹을 생성


## 1. Clone the example repository

- AWS EC2 인스턴스를 프로비저닝 하기 위한 얘시 configuration을 포함한 레포지토리를 복사하고 해당 폴더로 들어가면 다섯개 파일이 있다..

```cli
git clone https://github.com/hashicorp-education/learn-terraform-resources
```

1. `init-script.sh` : 의존성을 설치하고 샘플 PHP 어플리케이션을 시작하기 위한 프로비저닝 script
2. `terraform.tf` : configuration에 의해 요구되는 providers를 정의하는 terraform 블록
3. `main.tf` : 단일 EC2 인스턴스에 대한 configuration
4. `outputs.tf` : 리소스의 출력값에 대한 정의
5. `README.md` : 내용과 레포지토리에 대한 설명

- main.tf에 있는 두 리소스 블록 random_pet.name 과 aws_instance.web을 보기.

### 1.1 `random_pet` 리소스

첫 번째 리소스 블록은 `name`이라는 이름의 `random_pet` 리소스를 정의하며, 이 리소스는 무작위로 애완동물 이름을 생성합니다. 이 리소스에서 생성된 이름을 사용하여 다른 리소스의 이름이 모두 고유하도록 할 수 있습니다.

resource "random_pet" "name" {}

리소스 블록은 리소스 유형과 이름을 선언합니다. 유형과 이름은 함께 리소스 식별자(ID)를 구성하며, 형식은 resource_type.resource_name입니다. 이 경우 random_pet.name이 됩니다. 리소스 ID는 워크스페이스 내에서 고유해야 합니다. Terraform은 출력에서 ​​이 리소스에 대한 정보를 표시할 때 리소스 ID를 사용합니다.
리소스 유형은 항상 공급자 이름 뒤에 밑줄이 붙는 형식으로 시작합니다. random_pet 리소스 유형은 random 공급자에 속합니다.
Terraform 레지스트리에는 Terraform 프로바이더와 관련 리소스에 대한 문서가 있습니다. random_pet 문서 페이지를 열어보면 random 프로바이더 문서 아래에 해당 문서가 포함되어 있는 것을 알 수 있습니다. 이 페이지에는 random_pet 리소스에 대한 설명, 사용 예제, 인자 참조 및 속성 참조가 있습니다. 리소스는 인자, 속성 및 메타 인자를 가질 수 있습니다.

인수는 특정 리소스를 구성합니다. 따라서 많은 인수는 리소스별로 다릅니다. 인수는 공급자가 지정한 대로 필수 또는 선택적일 수 있습니다. 필수 인수를 제공하지 않으면 Terraform은 오류를 표시하고 구성을 적용하지 않습니다.
속성은 기존 리소스에서 노출되는 값입니다. 리소스 속성에 대한 참조는 `resource_type.resource_name.attribute_name` 형식을 취합니다. 인프라 객체의 구성을 지정하는 인자와 달리, 리소스의 속성은 종종 기본 클라우드 공급자 또는 API에 의해 할당됩니다.
메타 인수는 리소스의 동작을 변경합니다. 예를 들어, count 메타 인수를 사용하면 여러 리소스를 생성할 수 있습니다. 메타 인수는 Terraform 자체의 함수이며 리소스 또는 공급자에 종속되지 않습니다.

random_pet 리소스는 네 개의 선택적 인수를 가지며 하나의 속성을 노출합니다. 필수 인수가 없으므로 인수 없이 random_pet.name 리소스를 정의할 수 있습니다.


### 1.2 EC2 인스턴스 리소스

`aws_instance.web` 리소스 블록은 AWS EC2 인스턴스를 생성하기 위한 `web`이라는 이름의 `aws_instance` 리소스를 정의합니다.

resource "aws_instance" "web" {
  ami                    = "ami-a0cfeed8"
  instance_type          = "t2.micro"
  user_data              = file("init-script.sh")

  tags = {
    Name = random_pet.name.id
  }
}

aws_instance.web 리소스 블록 내부의 인수는 생성할 리소스 유형을 지정합니다. 

instance_type 및 ami 인수는 AWS 공급자에게 ami-a0cfeed8 머신 이미지를 사용하여 t2.micro EC2 인스턴스를 생성하도록 지시합니다.

참고: 이 구성은 us-west-2 리전에 배포됩니다. 다른 리전에 배포해야 하거나 AWS 계정에서 이 AMI에 액세스할 수 없는 경우, aws_instance 리소스의 ami 인수를 해당 리전에서 액세스할 수 있는 AMI로 업데이트해야 합니다. 다른 AMI를 사용하는 경우 init-script.sh 파일을 수정해야 할 수도 있습니다. EC2 인스턴스에 연결하여 /var/log/cloud-init-output.log 파일을 검토하여 오류를 디버깅하십시오.

user_data 인수는 file() 함수를 사용하여 init-script.sh의 내용을 반환합니다.

`tags` 인수는 이 EC2 인스턴스의 이름을 지정합니다. 이 인수는 `random_pet.name`의 ID 속성(`random_pet.name.id`)을 참조하여 EC2 인스턴스에 고유한 이름을 부여합니다. 이는 EC2 인스턴스와 `random_pet` 리소스 간의 암묵적인 종속성을 정의합니다. Terraform은 인스턴스에 이름이 지정될 때까지 인스턴스를 생성할 수 없습니다.

이는 사용 가능한 aws_instances 인수의 일부일 뿐입니다. 전체 목록은 aws_instance 설명서 페이지를 참조하십시오.

---

## 2. 인프라 생성



---

## 3. Associate security group with instance
## 4. 인프라 정리