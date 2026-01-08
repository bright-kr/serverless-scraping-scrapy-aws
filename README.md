# Scrapy 및 AWS로 서버리스 スクレイピング하기

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/) 

이 가이드는 Scrapy Spider를 작성하고, 이를 AWS Lambda에 배포하며, スクレイピング한 데이터를 S3 버킷에 저장하는 방법을 설명합니다. 더 빠르고 신뢰할 수 있는 솔루션을 원하십니까? [Bright Data's Serverless Functions](https://brightdata.co.kr/products/web-scraper/functions)는 70개 이상의 사전 구축된 JavaScript 템플릿, 클라우드 기반 IDE, 그리고 강력한 언블로킹 기능을 제공합니다. 가격은 [$2.7/1,000 page loads](https://brightdata.co.kr/pricing/web-scraper/functions)부터 시작하여, Webスクレイピング에 최적의 솔루션입니다.

- [Prerequisites](#prerequisites)
- [What is Serverless?](#what-is-serverless)
- [Getting Started](#getting-started)
- [Writing the Code](#writing-the-code)
- [Deploying To AWS Lambda](#deploying-to-aws-lambda)
- [Troubleshooting Tips](#troubleshooting-tips)

## Prerequisites

이 작업을 수행하려면 다음이 필요합니다:

- **Python에 대한 기본 이해**: 코드는 Python으로 작성합니다.
- **AWS (Amazon Web Services) Account**: AWS Lambda를 사용하므로 AWS 계정이 필요합니다.
- **WSL 2가 설치된 Linux 또는 Windows 머신**: Amazon은 코드를 실행하기 위해 _Amazon Linux_를 사용합니다. 코드를 업로드할 때 바이너리 호환성이 필요합니다.
- **Scrapy에 대한 기본 지식**: Scrapy로 スクレイピング하는 기본 지식이 도움이 됩니다.

## What is Serverless?

서버리스 아키텍처는 실제 사용량에 대해서만 비용을 청구하여 서버 관리 필요성을 제거합니다. 예를 들어, 스크레이퍼가 하루에 1분만 실행된다면 Lambda 같은 서비스가 항상 실행되는 서버보다 더 비용 효율적일 수 있습니다.

**Pros**
- **Billing:** 사용한 만큼만 지불합니다.
- **Scalability:** 자동 스케일링을 지원합니다.
- **Server Management:** 유지보수가 필요 없습니다.

**Cons**
- **Latency:** 콜드 스타트로 인해 실행이 지연될 수 있습니다.
- **Execution Time:** 최대 15분으로 제한됩니다.
- **Portability:** 벤더의 에코시스템에 종속됩니다.

## Getting Started

### Setting Up Services

AWS 계정에 로그인하고 S3 버킷을 생성합니다. 이를 위해 [‘All services’](https://us-east-2.console.aws.amazon.com/console/services) 페이지로 이동한 뒤 아래로 스크롤합니다.

![A list of all AWS services](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-108.png)

_Storage_ 섹션의 첫 번째 옵션인 _S3_를 클릭합니다.

![The S3 storage service](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-109.png)

다음으로 _Create bucket_ 버튼을 클릭합니다.

![Creating a new S3 bucket](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-110.png)

버킷 이름을 지정하고 설정을 선택합니다. 이 가이드를 따라 하는 목적이라면 기본 설정을 사용하셔도 됩니다.

![Naming the new S3 bucket and choosing your settings](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-113.png)

페이지 오른쪽 하단에 있는 _Create bucket_ 버튼을 클릭합니다.

![Creating the new S3 bucket](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-111.png)

새로 생성된 버킷은 _Amazon S3_ 아래의 _Buckets_ 탭에 표시됩니다.

![Clicking the create bucket button when the configuration is done](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-112.png)

### Setting Up Your Project

새 프로젝트 폴더를 생성합니다.

```bash
mkdir scrapy_aws
```

새 폴더로 이동한 뒤 가상 환경을 생성합니다.

```bash
cd scrapy_aws
python3 -m venv venv  
```

환경을 활성화합니다.

```bash
source venv/bin/activate
```

Scrapy를 설치합니다.

```bash
pip install scrapy
```

### What To Scrape

대상 사이트로 [books.toscrape](https://books.toscrape.com/)를 사용하겠습니다. 이 사이트는 Webスクレイピング 학습을 위해 만들어진 교육용 사이트입니다. 각 책은 class name이 `product_pod`인 `article`입니다. 우리는 페이지에서 이 모든 요소를 추출하려고 합니다.

![Inspecting one of the book in an article tag](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-114.png)

각 책의 제목은 `h3` 요소 안에 중첩된 `a` 요소 내부에 포함되어 있습니다.

![Inspecting the book title](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-115.png)

각 가격은 `div` 내부에 중첩된 `p`에 포함되어 있으며, class name은 `price_color`입니다.

![Inspecting the book price](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-116.png)

## Writing the Code

새 Python 파일 `aws_spider.py`를 열고, 다음 코드를 붙여 넣습니다:

```python
import scrapy


class BookSpider(scrapy.Spider):
    name = "books"
    allowed_domains = ["books.toscrape.com"]
    start_urls = ["https://books.toscrape.com"]

    def parse(self, response):
        for card in response.css("article"):
            yield {
                "title": card.css("h3 > a::text").get(),
                "price": card.css("div > p::text").get(),
            }
        next_page = response.css("li.next > a::attr(href)").get()

        if next_page:
            yield scrapy.Request(response.urljoin(next_page))
```

Spider를 테스트하면 가격이 포함된 책 목록으로 채워진 JSON 파일이 출력되어야 합니다:

```bash
python -m scrapy runspider aws_spider.py -o books.json
```

Spider를 실행하기 위한 핸들러를 두 개 만들겠습니다: 하나는 로컬에서 실행하는 용도, 다른 하나는 Lambda에서 실행하는 용도입니다.

다음은 로컬 핸들러 `lambda_function_local.py`입니다:

```python
import subprocess

def handler(event, context):
    # Output file path for local testing
    output_file = "books.json"

    # Run the Scrapy spider with the -o flag to save output to books.json
    subprocess.run(["python", "-m", "scrapy", "runspider", "aws_spider.py", "-o", output_file])

    # Return success message
    return {
        'statusCode': '200',
        'body': f"Scraping completed! Output saved to {output_file}",
    }

# Add this block for local testing
if __name__ == "__main__":
    # Simulate an AWS Lambda invocation event and context
    fake_event = {}
    fake_context = {}

    # Call the handler and print the result
    result = handler(fake_event, fake_context)
    print(result)
```

`books.json`을 삭제합니다. 다음 명령으로 로컬 핸들러를 테스트할 수 있습니다:

```bash
python lambda_function_local.py
```

모든 것이 올바르다면 프로젝트 폴더에 새로운 `books.json`이 생성됩니다.

다음은 Lambda용 핸들러입니다. 기본적으로 동일하게 동작하지만, 데이터를 S3 버킷에 저장합니다:

```python
import subprocess
import boto3

def handler(event, context):
    # Define the local and S3 output file paths
    local_output_file = "/tmp/books.json"  # Must be in /tmp for Lambda
    bucket_name = "aws-scrapy-bucket"
    s3_key = "scrapy-output/books.json"  # Path in S3 bucket

    # Run the Scrapy spider and save the output locally
    subprocess.run(["python3", "-m", "scrapy", "runspider", "aws_spider.py", "-o", local_output_file])

    # Upload the file to S3
    s3 = boto3.client("s3")
    s3.upload_file(local_output_file, bucket_name, s3_key)

    return {
        'statusCode': 200,
        'body': f"Scraping completed! Output uploaded to s3://{bucket_name}/{s3_key}"
    }
```

- 먼저 데이터를 임시 파일 `local_output_file = "/tmp/books.json"`에 저장합니다. 이는 데이터가 유실되는 것을 방지합니다.
- `s3.upload_file(local_output_file, bucket_name, s3_key)`로 버킷에 업로드합니다.

## Deploying To AWS Lambda

AWS Lambda에 배포해 보겠습니다. _package_ 폴더를 만듭니다:

```bash
mkdir package
```

의존성을 `package` 폴더로 복사합니다.

```bash
cp -r venv/lib/python3.*/site-packages/* package/
```

파일을 복사합니다. 로컬 핸들러가 아니라 Lambda용으로 만든 핸들러를 복사해야 합니다.

```bash
cp lambda_function.py aws_spider.py package/
```

package 폴더를 zip 파일로 압축합니다:

```bash
zip -r lambda_function.zip package/
```

ZIP 파일이 생성되면 AWS Lambda로 이동하여 _Create function._ 을 선택합니다. 안내에 따라 runtime(Python) 및 architecture 등 기본 정보를 입력합니다.

S3 버킷에 접근할 수 있는 권한을 부여하는 것도 잊지 마십시오.

![Creating a new function](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-117.png)

함수를 생성한 후, 소스 탭의 오른쪽 상단에 있는 _Upload from_ 드롭다운을 선택합니다.

![Lambda upload from](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-118.png)

_.zip file_을 선택하고 생성한 ZIP 파일을 업로드합니다.

![Uploading the created ZIP file](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-119.png)

_test_ 버튼을 클릭하고 함수가 실행될 때까지 기다립니다. 실행이 완료되면 S3 버킷을 확인하고, 새로운 파일 _books.json_이 생성되었는지 확인합니다.

![The new books.json file in your S3 bucket](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-120.png)

## Troubleshooting Tips

### Scrapy Cannot Be Found

Scrapy를 찾을 수 없다는 오류가 발생하면, `subprocess.run()`의 command array에 다음을 포함하십시오.

![Adding a piece of code in the subprocess.run() function](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-121.png)

### General Dependency Issues

Python 버전이 동일한지 확인해야 합니다. 로컬에 설치된 Python을 확인하십시오.

```bash
python --version
```

이 명령이 Lambda 함수와 다른 버전을 출력한다면, Lambda 구성을 이에 맞게 변경하십시오.

### Handler Issues

![The handler should match the function you wrote](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-122.png)

핸들러가 `lambda_function.py`의 함수와 일치하는지 확인하십시오. 예를 들어 `lambda_function.handler`는 `lambda_function`이 파일명이고 `handler`가 함수명임을 의미합니다.

### Can’t Write to S3

출력을 저장할 때 권한 문제가 발생하면, Lambda 인스턴스에 필요한 권한을 추가하십시오. 이를 위해 [IAM Console](https://console.aws.amazon.com/iam/)로 이동하여 Lambda 함수를 찾은 다음, _Add permissions_ 드롭다운을 클릭합니다.

_Attach policies_를 클릭합니다.

![Clicking on 'attach policies' in the Lambda function](https://github.com/luminati-io/serverless-scraping-scrapy-aws/blob/main/images/image-123.png)

_AmazonS3FullAccess_를 선택합니다.

![Selecting AmazonS3FullAccess](https://brightdata.co.kr/wp-content/uploads/2024/12/image-124-1024x214.png)

## Conclusion

수동 スクレイピング이 번거로우시다면, [Web Scraper APIs](https://brightdata.co.kr/products/web-scraper) 및 [ready-made datasets](https://brightdata.co.kr/products/datasets)를 확인해 보십시오. 지금 가입하고 무료 체험을 시작해 보십시오!