+++
title = 'coupang TV[1] - 백엔드'
date = 2024-08-07T06:51:55+09:00
categories = ["services"]
tags = ["flask", "coupang tv", "backend", "python"]
toc = true
+++

이번 글에서는 프로젝트 백엔드 부분을 소개한다. 스크래핑 기능, 데이터베이스 모델을 중점적으로 다룬다.

<!--more-->

## 스크래핑

1. 쿠팡에서 TV 정보 수집
2. 수집 정보 바탕으로 다나와에서 스펙 수집
3. 쿠팡 url -> 쿠팡 파트너스 url 변환

---

### 1. 쿠팡에서 TV 정보 수집

사용한 파이썬 라이브러리: `requests`, `bs4`

쿠팡은 처음 `url`을 어떻게 설정하느냐에 따라 검색 결과가 많이 달라진다. 때문에 디테일하게 설정해야한다.

#### 1.1 기본 URL

```
# 전체 url
https://www.coupang.com/np/search?rocketAll=true&q=tv&brand=6231%2C258%2C259%2C50444%2C3143%2C53215%2C49912%2C40385%2C46698%2C17260%2C75039%2C49609%2C72371%2C56188%2C19476%2C106528%2C3154%2C75097%2C17437%2C19237&filterType=rocket%2Crocket_luxury%2Crocket_wow%2Ccoupang_global&isPriceRange=true&priceRange=100000&minPrice=100000&maxPrice=2147483647&sorter=saleCountDesc&listSize={listSize}&page={page}
```

```
- https://www.coupang.com/np/
  - 쿠팡 기본 url

- search?rocketAll=true&q=tv
  - 검색시작, 로켓배송 필터, tv 검색

- brand=6231%2C258%2C259%2C50444%2C3143%2C53215%2C49912%2C40385%2C46698%2C17260%2C75039%2C49609%2C72371%2C56188%2C19476%2C106528%2C3154%2C75097%2C17437%2C19237
  - 브랜드 설정, 모든 브랜드 선택

- filterType=rocket%2Crocket_luxury%2Crocket_wow%2Ccoupang_global&isPriceRange=true&priceRange=100000&minPrice=100000&maxPrice=2147483647
  - filterType은 자동 설정됨, 가격 10만원 이상 필터(10만원 이하로 내려가면 tv 부자재가 나옴)

- sorter=saleCountDesc&listSize={listSize}&page={page}
  - 판매량 내림차순 정렬, 리스트사이즈 설정, 페이지 설정
    - 리스트 사이즈는 100(max)로 설정해서 최소한의 트래픽 일으킴
    - 페이지는 1부터 일정 조건 도달할 때까지 루프
```

#### 1.2 Header

User-Agent와 Accept-Language를 설정해줘야한다. 특히 Accept-Language를 설정하지 않으면 리퀘스트 응답이 안온다.

```python
headers = {
            "User-Agent": "your user agent",
            "Accept-Language": "en-US,en;q=0.9,ko-KR;q=0.8,ko;q=0.7",
        }
```

#### 1.3 기본 URL에서 수집하는 정보

기본 URL에서는 다음의 항목을 수집한다.

1. 제품명
2. 모델명
3. 설치 방법(방문설치 or 자가설치)
4. tv 유형(벽걸이형 or 스탠드형)
5. 제품 사진
6. 리뷰 점수, 리뷰 수

일반적인 경우 제품명에 모델명, 설치방법, tv 유형이 포함되어있다.

> 샤오미 안드로이드11 4K UHD LED A Pro TV, 165cm(65인치), L65M8-A2KR, 스탠드형, 방문설치

split function을 활용해서 각각의 정보를 찾는데, 해당 단어가 제품명에 포함되어있는지 확인하면 되는 설치 방법이나 tv 유형과 달리 `모델명`은 단순히 포함 유무를 가지고 판단할 수 없기 때문에 `정규표현식`을 활용하여 수집한다.

##### 활용된 정규표현식

```python
# Define the pattern to match the model names
pattern = re.compile(r"^(?=.*[A-Z])(?=.*[0-9])[A-Z0-9]+(-[A-Z0-9]+)?(\(.*\))?$")
pattern2 = re.compile(
    r"\b[A-Za-z]{1,2}\d{2,}([A-Za-z0-9]*[A-Za-z]{1,2}[A-Za-z0-9]*)?\b"
)
```

##### 정규표현식을 활용한 모델명 구하는 함수

```python
def get_model_name(name):
    """
    function name = get_model_name
    params =
        div tag
    explain = This function get model name from product title.
        ex) PRISM 4K UHD TV, 139.7cm(55인치), PTC550UD, 스탠드형, 방문설치 -> PTC550UD
    """
    name_split = name.split(",")
    for n in name_split:
        n = n.strip()
        if pattern.search(n):
            model_name = n
            break
        else:
            model_name = "no model name"

    if model_name == "no model name":
        name_split = name.split()
        for n in name_split:
            n = n.strip()
            if pattern.search(n):
                model_name = n
                if len(model_name) < 3:
                    continue
                break
            else:
                model_name = "no model name"

    # Check if model_name is "no model name"
    if model_name == "no model name":
        match = pattern2.search(name)
        if match:
            model_name = match.group()

    if name.startswith("TCL") and len(model_name) < 4:
        try:
            match = size_pattern.search(name)
            if match:
                size = match.group()
                if len(size) == 4:
                    size = size[:2]
                else:
                    size = size[:3]
                model_name = size + model_name
        except:
            pass

    return model_name

```

패턴 1을 통해 1차로 모델명을 구한 뒤, 모델명을 구하지 못한 상품들을 대상으로 패턴 2를 통해 2차로 모델명을 구한다.
TCL 브랜드 모델명의 경우 쿠팡 제품명 속 모델명이 3글자가 되지 않는 경우가 있었고 `ex) V6B` 정식 모델명은 사이즈 + 모델명이었기 때문에 `ex) 43V6B` 예외 처리하여 따로 구했다.

#### 1.4 제품 상세 URL 에서 수집하는 정보

1. 기본 가격
2. 할인 가격
3. 쿠폰 가격
4. 브랜드명
5. 짧은 제품명
6. 제품 URL

제품 상세 url은 기본 URL에서 수집한 상품 리스트에서 추출한다. 쿠팡 가격에는 크게 3가지 가격이 있는데, 위의 1, 2, 3번이 그것이다.

```
# 기본적으로 다음과 같다.
기본 가격 > 할인 가격 > 쿠폰 가격
```

브랜드명은 제품명 제일 앞에 위치하지만, 쿠팡 상세 제품 페이지에는 브랜드명을 기입할 수 있는 별도의 공간이 있다. 그곳에 브랜드명이 기입되어있지 않다면 `중소기업`으로 분류했다.

---

#### 1.5 기본 정보 데이터베이스 저장

```python
# scraper_basic.py 일부.
product_data = {
    "name": name,
    "short_name": short_name,
    "model_name": model_name,
    "install_info": install_info,
    "tv_type": tv_type,
    "product_pic_s": product_pic_s,
    "rating": rating,
    "rating_count": rating_count,
    "original_price": original_price,
    "sale_price": sale_price,
    "coupon_price": coupon_price,
    "highest_price": highest_price,
    "lowest_price": lowest_price,
    "discount_rate": discount_rate,
    "brand": brand,
    "product_url": product_url,
}

update_or_create_product(product_data)
```

위의 과정을 통해 모은 데이터를 데이터베이스에 저장한다. 이미 데이터베이스에 해당 제품이 있을 경우에는 가격등 정보를 업데이트하고 제품이 없을 경우에는 새로운 instance를 추가한다.

---

### 2. 수집 정보 바탕으로 다나와에서 스펙 수집

쿠팡에도 tv 제품에 대한 스펙 사항이 표로 정리되어 있기는 하지만 제품마다 다루는 종목이 상이하다. 쿠팡 상품 상세 설명에 있는 이미지를 LLM을 활용해 해석해서 특성을 추출할까했지만 일이 너무 복잡해져서 그 방법은 보류했다.

대신 [다나와](https://search.danawa.com/dsearch.php)에서 모델명을 검색해서 제품 스펙 사항을 수집했다. 다나와도 마찬가지로 스펙 종목이 제품마다 상이했지만 그 수가 많고 반복적이었기 때문에 데이터베이스에 정리하기 용이했다.

사용한 파이썬 라이브러리: `requests`, `bs4`

#### 2.1 제품 스펙 데이터 수집 및 전처리

제품 스펙 데이터 자체는 얻는데 문제가 없었지만, 잘 정리되어있지 않았기 때문에 전처리하는 과정이 필요했다.

{{< image_caption src="spec_ex.png" alt="스펙 이미지 예시" caption="스펙 이미지 예시" >}}

```html
<!-- 제품 스펙 html 예시) 모델명 KQ65LSB01AFXKR의 경우-->
<div class="spec_list">
  <a
    href="#"
    class="view_dic"
    onclick="$.termDicViewLink(31510, 'view', this, 0, 149, 180); return false;"
    ><span class="cm_mark">QLED TV</span></a
  ><em>/</em><span class="cm_mark">65인치(163cm)</span><em>/</em
  ><a
    href="#"
    class="view_dic"
    onclick="$.termDicViewLink(228906, 'view', this, 0, 149, 180); return false;"
    >4K UHD</a
  ><em>/</em
  ><a
    href="#"
    class="view_dic"
    onclick="$.termDicViewLink(382, 'view', this, 0, 149, 180); return false;"
    ><span class="cm_mark">60Hz</span></a
  ><em>/</em><br /><span class="cm_mark">[화질]</span> 퀀텀4K<em>/</em
  ><a
    href="#"
    class="view_dic"
    onclick="$.termDicViewLink(219549, 'view', this, 0, 149, 180); return false;"
    >HDR10+</a
  ><em>/</em>
  ...
</div>
```

table로 정리되어있지않다. 그리고 html을 soup로 변환시켜서 해당 div의 text를 가져왔을 때 스펙 사항만 깔끔하게 가져오지는 못해서 전처리가 필요했다.

```python
# scrape_feautre.py 일부
spec_list = spec_list.text
spec_list = spec_list.split("/")
clean_spec_list = [
    spec.strip().replace("\t", "").replace("\n", "") for spec in spec_list
]
# Initialize the dictionary
dic = {
    "기본": {
        "screen_size": "",
        "display_tech": "",
        "resolution": "",
        "refresh_rate": "",
    },
    "화질": {},
    "사운드": {},
    "스마트": {},
    "부가": {},
    "보증기간": {},
    "게임": {},
}

# Populate the dictionary
current_category = "기본"
for spec in clean_spec_list:
    if "[화질]" in spec:
        current_category = "화질"
    elif "[게임]" in spec:
        current_category = "게임"
    elif "[스마트]" in spec:
        current_category = "스마트"
        dic[current_category][spec.replace("[스마트] ", "")] = True
    elif "[사운드]" in spec:
        current_category = "사운드"
    elif "[부가]" in spec:
        current_category = "부가"
    elif "[보증기간]" in spec:
        current_category = "보증기간"

    if current_category == "기본":
        if "인치" in spec:
            dic["기본"]["screen_size"] = extract_screen_size(spec)
        elif spec.endswith("ED"):
            dic["기본"]["display_tech"] = spec
        elif spec.endswith("HD"):
            dic["기본"]["resolution"] = spec
        elif "주사율" in spec:
            dic["기본"]["refresh_rate"] = spec.split(": ")[1]

    else:
        if "]" in spec:
            spec = spec.split("]")[1].strip()
        if ":" in spec:
            spec_split = spec.split(":")
            k = spec_split[0].strip()
            v = spec_split[1].strip()
            if v == "X":
                continue
            dic[current_category][k] = v
            continue

        if len(spec) > 20:
            continue
        dic[current_category][spec] = True

print(f"model name : {model_name}\n spec list: {clean_spec_list}")
del dic["보증기간"]

for cat in dic:
    for feature in dic[cat]:
        if feature == "크기(가로x세로x깊이)":
            continue
        code = codes[cat]
        check_and_add_feature_name(
            name=feature,
            value=dic[cat][feature],
            feature_code=code,
            prod=product,
        )

```

우선 em tag의 '/'를 기준으로 스펙 사항을 분리한다. 그리고 불필요한 요소를 제거한 clean_spec_list를 만든다.
그리고 스펙리스트 하나하나의 사항을 for 문을 통해 어떤 구분에 대한 스펙인지 분류하고, 데이터베이스에 저장하기위한 전 작업을한다.

오직 다나와의 지저분한 정보를 수집하기위해 재활용 가능성이 낮은 코드를 작성했지만, 그래도 깨끗하지않은 정보를 나름대로 정리할 수 있었다.

---

### 3. 쿠팡 url -> 쿠팡 파트너스 url 변환

사용한 파이썬 라이브러리: `selenium`, `dotenv`, `undetected_chromedriver`

제품 url을 그대로 가져와서 옮기기만 하면 쿠팡 파트너스로서 수익을 창출할 수 없다. 오직 쿠팡 파트너스 페이지를 통해 변환된 url을 통해 유입되어야 그 유입으로 인해 매출이 발생했을 때 수수료를 얻는다.

쿠팡 파트너스에서 파트너스 API를 통해 제품 url을 파트너스 url로 변환하는 서비스를 제공하긴 하지만 일정 수준 이상 수익이 발생한 파트너 중 쿠팡의 심사에 통과한 파트너에게만 제공한다.

{{< image_caption src="partners_api.png" alt="파트너스 api" caption="파트너스 api 활용 docs" >}}

#### 3.1 심사에 통과하지 못한 자의 링크 생성

원래는 모델명을 검색해서 나오는 리스트 중 내 제품과 일치하는 제품을 찾아서 일일히 링크를 생성해야했다. 링크를 생성하는 과정에서 정확히 같은 제품이어야했기 때문에 제품 이미지 정보를 활용해서 같은 이미지 소스를 가지고 있는 제품을 선택하곤 했다. 하지만 같은 이미지를 여러 다른 모델에 활용하는 경우도 있는 등 애로사항이 발생하던 차에 간편 링크 만들기 기능이 추가되었다. 간편 링크 만들기 기능은 상품의 url을 입력하면 파트너스 url로 변환하여 제공한다.
{{< image_caption src="link_ex.png" alt="간편 링크 생성 예시" caption="간편 링크 생성 예시" >}}

이 기능을 selenium을 통해 자동화하여 약 500개의 파트너스 api를 얻어보고자 한다.

#### 3.2 링크 생성 자동화 - 로그인

쿠팡 파트너스 로그인을 위해 프로젝트 root 디렉토리에 .env file을 만들어 로그인 정보를 저장한다. 저장된 .env 파일이 깃헙에 업로드되면 안되기 때문에 .gitignore 파일에 .env를 추가한다.

```.env
ID="your coupang email"
PW="your coupang password"
```

그리고 .env 정보를 활용하여 쿠팡에 로그인한다.

```python
# scraper_url2.py의 일부

# Load environment variables from .env file
load_dotenv()

# Get ID and password from environment variables
user_id = os.getenv("ID")
user_password = os.getenv("PW")

...

def login_coupang():
    # Login to Coupang partners
    login_input = wait.until(
        EC.presence_of_element_located((By.ID, "login-email-input"))
    )
    password_input = wait.until(
        EC.presence_of_element_located((By.ID, "login-password-input"))
    )
    submit_button = wait.until(
        EC.element_to_be_clickable(
            (
                By.CSS_SELECTOR,
                ".login__button.login__button--submit._loginSubmitButton.login__button--submit-rds",
            )
        )
    )

    login_input.send_keys(user_id)
    password_input.send_keys(user_password)
    submit_button.click()
```

#### 3.3 링크 생성 자동화 - 모달 대응

로그인 후 링크를 수집하다보면 200~300개를 수집했을 때 쯤 비밀번호를 다시 입력하라는 모달이 나오면서 스크래핑을 방해한다. 이에 대응하기 위해 아래와 같은 함수를 만들어서 스크래핑 과정 중간중간에 함수를 호출했다.

```python
# scraper_url2.py 중 일부

# Function to handle the modal
def handle_modal():
    try:
        modal_present = driver.find_element(By.CLASS_NAME, "ant-modal-content")
        if modal_present:
            password_input = driver.find_element(By.ID, "password")
            password_input.send_keys(user_password)
            confirm_button = driver.find_element(
                By.XPATH, "//button[@type='submit']"
            )
            confirm_button.click()
            # Wait until modal disappears
            WebDriverWait(driver, 10).until(
                EC.invisibility_of_element_located(
                    (By.CLASS_NAME, "ant-modal-content")
                )
            )
            print("modal handled@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@")
    except Exception as e:
        # print("No modal present or error handling modal")
        pass
```

모달이 없을 때에는 try except 문에서 except가 발동되면서 아무일도 일어나지 않고, 모달이 있을 때에는 try가 완료되면서 모달에 비밀번호를 입력한다.

#### 3.4 링크 생성 자동화 - 늦은 url / 반복 url 대응

링크를 반복적으로 빠르게 생성하다보면 어떤 이유에서인지 url이 늦게 생성되거나 그 전의 제품과 같은 url로 미리 값이 입력되어있어 잘못된 정보를 수집하는 경우가 있었다. 이를 방지하기위해 그 전의 제품 파트너스 url과 지금 생성된 파트너스 url이 다른지 확인하는 함수를 만들었다.

```python
# scraper_url2.py 중 일부

# Define a custom wait condition
def text_is_not_none(driver):
    shorten_url = wait.until(
        EC.presence_of_element_located(
            (
                By.CSS_SELECTOR,
                ".unselectable-input.tracking-url-input.large",
            )
        )
    )
    condition = True
    while condition:
        element = driver.find_element(
            By.CSS_SELECTOR, ".unselectable-input.tracking-url-input.large"
        ).text.strip()

        # print(f"current url = {element}, previous url = {previous_url}")
        if element == previous_url:
            handle_modal()
            print("repeat link error. try again...")
            time.sleep(3)
        else:
            condition = False

    return element
```

#### 3.5 수집한 파트너스 url 데이터베이스 저장

```python
for i in range(1, len(products) + 1):
    try:
        the_product = products[i - 1]
        # the_product.shorten_url = None

        print(the_product.id, the_product.model_name)
        if the_product.shorten_url:
            print(
                f"{the_product.model_name} already got shorten_url: {the_product.shorten_url}"
            )
            continue
        else:
            handle_modal()
            shorten_url_text = get_shorten_url(the_product, driver)

        # Introduce a random delay between 1 and 3 seconds
        time.sleep(random.uniform(1, 3))

        # Update the Product with the shorten_url
        if the_product:
            the_product.shorten_url = shorten_url_text
            db.session.commit()
```

만약 제품이 이미 파트너스 url(shorten_url)이 있다면 패스하고, 없다면 파트너스 url을 수집했다. 수집한 url은 위와 같이 데이터베이스에 저장한다.

---

---

## 데이터베이스 모델

1. 제품 테이블
2. 제품 특성 테이블
3. 제품 과거 가격 테이블
4. 데이터 전처리 함수

---

### 1. 제품 테이블

제품 테이블 필드 구성을 소개한다.

```python
# models.py 일부
class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(255), nullable=False)
    short_name = db.Column(db.String(255), nullable=False)
    model_name = db.Column(db.String(255), nullable=False)
    install_info = db.Column(db.String(50), nullable=False)
    tv_type = db.Column(db.String(50), nullable=False)
    product_pic_s = db.Column(db.String(255), nullable=False)
    rating = db.Column(db.String(50), nullable=False)
    rating_count = db.Column(db.String(50), nullable=False)
    original_price = db.Column(db.Float, nullable=True)
    sale_price = db.Column(db.Float, nullable=True)
    coupon_price = db.Column(db.Float, nullable=True)
    highest_price = db.Column(db.Float, nullable=True)
    lowest_price = db.Column(db.Float, nullable=True)
    discount_rate = db.Column(db.Float, nullable=True)
    brand = db.Column(db.String(50), nullable=False)
    product_url = db.Column(db.String(255), nullable=False)
    shorten_url = db.Column(db.String(255), nullable=True)
    last_checked = db.Column(db.Date, nullable=False, default=date.today)
    features_scraped = db.Column(db.Boolean, nullable=False, default=False)
    best_offer = db.Column(db.Boolean, nullable=False, default=False)

    # Unique constraint
    __table_args__ = (
        db.UniqueConstraint(
            "name", "model_name", "tv_type", name="_name_model_tvtype_uc"
        ),
    )

    # Relationship with Feature
    features = db.relationship(
        "Feature",
        secondary=product_features,
        backref=db.backref("products", lazy=True),
    )
    # Relationship with ProductLowestPriceHistory
    lowest_price_history = db.relationship(
        "ProductLowestPriceHistory", backref="product", lazy=True
    )
```

솔직히 크게 자신은 없다. 특히 필드 속성을 best로 설정했는지는 모르겠고, **table_args**를 활용해서 unique constraint를 만들긴 했지만 따로 활용하지는 않았다.

---

### 2. 특성 테이블

```python
# models.py 일부
class Feature(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    feature_code = db.Column(db.Integer, nullable=False)
    name = db.Column(db.String(255), nullable=False)
    value = db.Column(db.String(255), nullable=False)
    description = db.Column(db.Text, nullable=True)

# Association table for many-to-many relationship between Product and Feature
product_features = db.Table(
    "product_features",
    db.Column("product_id", db.Integer, db.ForeignKey("product.id"), primary_key=True),
    db.Column("feature_id", db.Integer, db.ForeignKey("feature.id"), primary_key=True),
)
```

{{< image_caption src="feature_table.png" alt="특성 테이블 구상표" caption="특성 테이블 구상표" >}}
위의 표와 같은 형식으로 정리하기 위해 feature 테이블을 구성했다.

#### 2.1 특성 데이터베이스 저장 방법

기본적으로 특성을 먼저 저장한 뒤에 저장한 특성을 제품과 연결한다. 특성을 저장할 때 이미 있는 특성인지 파악한 뒤에 이미 있는 특성인 경우 곧바로 제품과 해당 특성을 연결한다. 해당 과정을 수행하는 함수는 다음과 같다.

```python
# utils.py

def check_and_add_feature_name(name, feature_code, value, prod):
    # Check if the feature already exists
    existing_feature = Feature.query.filter_by(name=name, value=value).first()

    if existing_feature:
        print(f"Feature name: '{name}', value: '{value}' already exists.")

        # Check if the product already has this feature
        if existing_feature not in prod.features:
            prod.features.append(existing_feature)
    else:
        # Create a new feature if it doesn't exist
        new_feature = Feature(name=name, value=value, feature_code=feature_code)
        db.session.add(new_feature)
        db.session.commit()
        print(f"Feature '{name}', value: '{value}' added to the database.")
        prod.features.append(new_feature)

    # Commit the changes to the database
    db.session.commit()
```

---

### 3. 제품 과거 가격 테이블

```python
# models.py 중 일부

class ProductLowestPriceHistory(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    product_id = db.Column(db.Integer, db.ForeignKey("product.id"), nullable=False)
    date = db.Column(db.Date, nullable=False)
    lowest_price = db.Column(db.Float, nullable=False)
```

해당 날짜의 가장 낮은 가격(판매가)를 기록한다. 제품 id를 외래키로 갖는다.

---

### 4. 데이터 전처리 함수

기타 데이터베이스 운영을 위한 함수를 소개한다.

#### 4.1 없는 제품 삭제 함수

```python
# update_or_create_product.py

def delete_old_products():
    today = date.today()
    # Delete products that were not checked today
    Product.query.filter(Product.last_checked != today).delete()
    Product.query.filter(Product.features_scraped == 0).delete()
    db.session.commit()
```

매일 업데이트 하는 과정에서 당일 획득되지 않은 제품을 삭제한다. 이는 과거에는 쿠팡 제품 리스트에 있었지만 품절, 삭제 등의 이유로 수집되지 않은 제품을 포함한다.
하루만이라도 없으면 제품 자체를 삭제하는 것은 좀 과하다고 판단이 되서 추후 수정 예정이다.

#### 4.2 최고 조건 유지 함수

```python
# utils.py 중 일부

def isit_best_offer():
    # Step 2: Query to find all unique (model_name, tv_type) combinations
    product_groups = (
        db.session.query(Product.model_name, Product.tv_type).distinct().all()
    )

    for group in product_groups:
        model_name, tv_type = group

        # Step 3: Find the product with the lowest price in each group
        lowest_priced_product = (
            db.session.query(Product)
            .filter_by(model_name=model_name, tv_type=tv_type)
            .order_by(Product.lowest_price)
            .first()
        )

        # Step 4: Set best_offer to True for the lowest priced product
        if lowest_priced_product:
            lowest_priced_product.best_offer = True

            # Set best_offer to False for other products in the group
            other_products = (
                db.session.query(Product)
                .filter(
                    Product.id != lowest_priced_product.id,
                    Product.model_name == model_name,
                    Product.tv_type == tv_type,
                )
                .all()
            )

            for product in other_products:
                product.best_offer = False

    # Commit the changes to the database
    db.session.commit()
```

같은 제품인데 다른 판매자가 판매하는 경우, 최저가 제품만 리스팅하기 위해 제품에 best_offer 필드를 만들었다. 서비스에서 제공할 때 best_offer가 true 값을 갖는 제품만 리스팅한다.

#### 수정 필요 사항

...

#### 같은 주제 게시글 모음

{{< related_posts tag="coupang tv" >}}
