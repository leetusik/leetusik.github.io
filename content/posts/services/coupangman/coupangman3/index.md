+++
title = 'coupang TV[3] - 프론트엔드'
date = 2024-08-08T11:07:37+09:00
categories = ["services"]
tags = ["flask", "coupang tv", "frontend", "javascript", "jinja2"]
toc = true
+++

이번 글에서는 프로젝트 프론트엔드 부분을 소개한다. html, css, js, 그리고 css 프레임워크로 pico css를 사용한다. 또한 flask application이기 때문에 jinja2를 활용한다.

<!--more-->

<!-- ## 후기

1. 나중에는 flask와 호환이 좋은 bootstrap을 쓰거나 아니면 bulma, tailwind css 같은 좀 더 묵직한 css 프레임워크를 써보고 싶다.
2. 404 에러 등 에러 처리 라우트가 안됐다. 나중에 추가해야지. -->

## 홈페이지 소개

우리집 노트북 서버로 홈페이지를 운영하고있다.

> http://ggulmae.com/

### 1. How it looks

#### 1.1 메인 페이지

{{< image_caption src="mainpage.png" alt="메인 이미지" caption="메인 이미지" >}}

1. 왼쪽의 `사이드바`에서 각종 `필터`를 적용할 수 있다.
2. `이름`에서 제품의 사진과 함께 간단한 제품의 스펙을 확인할 수 있다.
3. `가격`에서 제품의 할인되기 전 가격과 할인 후 가격, 할인율을 확인할 수 있다.
4. `리뷰`에서 리뷰 점수, 리뷰 수를 확인할 수 있다.
5. `주요` 특징에서 스마트, 게임 기능을 지원하는지, 스탠드 or 벽걸이 형인지, 설치는 어떻게 진행되는지, 제품 생산 연도를 알 수 있다.

#### 1.2 상세 페이지

제품 상세 페이지는 메인 페이지에서 제품의 `이름`부분을 클릭해서 접근할 수 있다.
{{< image_caption src="detailpage.png" alt="제품 상세 페이지" caption="제품 상세 페이지" >}}

1. `왼쪽 상단`에서 제품의 이미지, 모델명, 현재 가격을 알 수 있다.
2. `오른쪽 상단`에서 제품의 지난 90일 가격 추이를 확인할 수 있다.
3. `나머지` 부분에서 제품의 기본 정보, 화질, 스피커, 스마트 기능, 게임 기능 등을 확인할 수 있다.

---

## Flask Things

### 1. routes.py

routes.py에서 메인 페이지와 상세 페이지를 모두 관리한다.

```python
# routes.py 중 일부

@main.route("/", methods=["GET"])
def index():
  ...

@main.route("/model/<int:id>", methods=["GET"])
def model(id):
  ...
```

#### 1.1 @main.route("/", methods=["GET"])

홈페이지에 사이드바에 필터 가능 제품 특징을 가져오기 위해 routes.py에서 필터 가능한 제품 특징을 리스트로 저장해서 index.html을 렌더링할 때 제공한다.

지금 다시보니 매번 index.html을 보여줄 때마다 모든 특징을 가져오는 것은 비효율적으로 보인다. 매일 새로 업데이트할 때마다 한 번 하면 좋을 것 같다.

```python
# routes.py 중 일부

sizes = get_screen_sizes()
panels = get_display_techs()
resolutions = get_resolutions()
brands = db.session.query(Product.brand).distinct().all()
brands = [brand[0] for brand in brands]  # Unpack tuples
release_years = get_release_years()
```

다음은 `@main.route("/")`가 리턴하는 값이다.

seleted\_가 붙은 변수들은 사용자가 메인 페이지에서 필터를 사용했을 경우, `requests.args.get("somthing", "")`함수로 받아온 선택된 필터 값이다. 이것을 활용해서 필터링된 값을 메인 페이지에서 제공한다.

```python
# routes.py 중 일부

return render_template(
    "index.html",
    products=products,
    brands=brands,
    selected_brands=selected_brands,
    sizes=sizes,
    selected_sizes=selected_sizes,
    resolutions=resolutions,
    selected_resolutions=selected_resolutions,
    panels=panels,
    selected_panels=selected_panels,
    selected_others=selected_others,
    release_years=release_years,
    selected_release_years=selected_release_years,
)
```

필터가 하나가 설정되면 다른 부분의 필터가 무시되는 에러가 있었는데, 챗지피티의 도움을 받아 다음과 같은 코드로 해결했다.

```python
# routes.py 중 일부

from sqlalchemy.orm import aliased

# Aliases for the Feature table to handle different filter criteria
size_feature = aliased(Feature)
panel_feature = aliased(Feature)
resolution_feature = aliased(Feature)
release_year_feature = aliased(Feature)
smart_tv_feature = aliased(Feature)
game_mode_feature = aliased(Feature)

```

#### 1.2 @main.route("/model/&lt;int:id&gt;", methods=["GET"])

`product_id`를 활용해서 제품과 제품의 특징을 가져오고, 마찬가지로 id를 활용해서 ProductLowestPriceHistory 데이터베이스에 접근해 dataframe 형식으로 제품의 과거 가격 값을 가져온 후 dictionary 형식으로 model.html을 호출할 때 리턴한다.

```python
@main.route("/model/<int:id>", methods=["GET"])
def model(id):
    product = Product.query.get_or_404(id)
    features = get_product_features(id)
    df = get_price_history(id)
    data = {
        "dates": [(date - df.index[0]).days for date in df.index],
        "prices": df["price"].tolist(),
    }
    return render_template(
        "model.html",
        product=product,
        features=features,
        price_history=data,
    )
```

### 2. Jinja2 Funtions

`__init__.py`의 `create_app`함수에 들어있는 진자 커스텀 필터를 소개한다.

```python
# __init__.py 중 일부

    # Custom filter to strip leading and trailing whitespace
    def strip_filter(value):
        if isinstance(value, str):
            return value.strip().replace("(", "").replace(")", "")
        return value

    # Custom filter to truncate text
    def truncate_filter(value, length=30):
        if isinstance(value, str) and len(value) > length:
            return value[:length] + "..."
        return value

    # Custom filter to match regex patterns
    def match_filter(s, pattern):
        return re.match(pattern, s) is not None

    # Custom filter to add commas as thousands separators
    def add_commas(value):
        value = int(value)
        if isinstance(value, (int, float)):
            return "{:,}".format(value)
        return value

    # Register the custom filter with Jinja
    app.jinja_env.filters["add_commas"] = add_commas
    app.jinja_env.filters["strip"] = strip_filter
    app.jinja_env.filters["truncate"] = truncate_filter
    app.jinja_env.filters["match"] = match_filter
```

- `strip_filter`는 리뷰수가 (123개) 형식으로 데이터베이스에 저장되어서 ()를 지우기 위해 추가했다.
- `truncate_filter`는 제품명이 메인 페이지에 표기될 때 길이를 30 이하로 유지시킨다.
- `match_filter`는 `index.html` if문에서 regex로 특징을 선별하기 위해서 사용한다.
- `add_commas`는 `model.html`에서 콤마 없이 숫자로만 저장되어있는 가격 데이터를 보기 좋게 변환하는데 사용한다.

jinja2 function을 추가하면서 백엔드에서 데이터를 수집할 때 제대로 하면 굳이 새로운 function을 추가하지 않아도 괜찮겠다는 생각을 했고, 또 어떤 것은 js로, 어떤 것은 jinja2로 해결을 하니 헷갈리는 부분이 많아서 어려움이 있었다. 웬만하면 백엔드에서 수집할 때 확실하게 잘 하고, 부족한 부분은 js로 처리해야겠다는 생각을 했다.

### 3. index.html

flask 기능인 url_for을 활용하니 파일 위치에 따라 잘못 불러오거나 하는 문제가 발생하지 않아서 좋았다.

```html
<!-- index.html 중 일부-->
<link
  rel="stylesheet"
  href="{{ url_for('static', filename='css/style.css') }}"
/>

...

<form
  method="GET"
  action="{{ url_for('main.index') }}"
  onsubmit="concatenateAll()"
>
  ...
</form>
```

#### 3.1 sidebar

`aside` 태그를 활용해서 메인 페이지 왼쪽에 사이드바를 만들었다. `aside` 태그 안에 `form`을 넣고, 그 안에 `fieldset`을 여러개 넣어서 여러 필터를 사용할 수 있게 만들었다. 다음은 필드셋 중 첫번째 브랜드의 필드셋이다.

`for 문`을 활용해서 모든 브랜드들에 대한 체크박스를 만들었고, 브랜드가 이미 url을 통해 `selected_brands`에 포함되어있는지 `if 문`을 통해 확인한 뒤, 이미 포함되었다면 체크박스에 체크를 한다.

```html
<!-- index.html 중 일부-->

<fieldset>
  <details>
    <summary>브랜드</summary>
    <div>
      <div>
        <input
          id="select-all-brand"
          type="checkbox"
          onclick="toggleSelectAllBrand(this)"
        />
        <label for="select-all-brand">All</label>
      </div>
      {% for brand in brands %}
      <div class="filter-container">
        <input
          id="brand-{{ loop.index }}"
          data-name="brand"
          type="checkbox"
          value="{{ brand }}"
          {%
          if
          brand
          in
          selected_brands
          %}checked{%
          endif
          %}
        />
        <label for="brand-{{ loop.index }}" title="{{ brand }}"
          >{{ brand }}</label
        >
      </div>
      {% endfor %}
    </div>
  </details>
</fieldset>
```

#### 3.2 main-content div

`thead`를 다음과 같이 구성하고 `for 문`을 통해 table item들을 추가했다.

```html
<thead>
  <tr>
    <th scope="col" class="product-name">이름</th>
    <th scope="col" id="price-header" class="sortable">
      가격 <span id="price-sort-icon"></span>
    </th>
    <th scope="col" id="rating-header" class="sortable">
      리뷰 <span id="rating-sort-icon"></span>
    </th>
    <th scope="col">주요 특징</th>
  </tr>
</thead>
```

제품의 이름 부분을 클릭했을 때 다음과 같은 코드로 제품 상세 페이지로 넘어가도록 설정했다.

> `href="{{ url_for('main.model', id=product.id) }}"`

### 4. model.html

차트를 그리기 위해서 다양한 패키지를 사용해야했다.

```html
<!--model.html 중 일부-->

<script src="https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.29.1/moment.min.js"></script>
<!-- Include Chart.js -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<!-- Include Chart.js date adapter for Moment.js -->
<script src="https://cdn.jsdelivr.net/npm/chartjs-adapter-moment@1.0.0"></script>
```

다음은 `routes.py`에서 받은 price_history 변수를 활용하여 차트를 만드는 script이다. 이는 html `canvas` tag에서 받아 입력된 값을 출력한다.

```html
<!--model.html 중 일부-->

<script>
  document.addEventListener('DOMContentLoaded', (event) => {
    const priceHistory = {{ price_history|tojson }};

    const labels = priceHistory.dates.map(days => {
      const date = new Date();
      date.setDate(date.getDate() - (89 - days));
      return date.toISOString().split('T')[0];
    });

    const minPrice = Math.min(...priceHistory.prices);
    const maxPrice = Math.max(...priceHistory.prices);

    // Convert minPrice to string, leave first digit unchanged, and turn the remaining digits to zero
    const minPriceStr = minPrice.toString();
    const firstDigit = minPriceStr[0];
    const zeros = '0'.repeat(minPriceStr.length - 1);
    const yMin = parseFloat(firstDigit + zeros);

    const ctx = document.getElementById('priceHistoryChart').getContext('2d');
    new Chart(ctx, {
      type: 'line',
      data: {
        labels: labels,
        datasets: [{
          label: '', // Set label to an empty string to hide it from the legend
          data: priceHistory.prices,
          borderColor: 'rgba(75, 192, 192, 1)',
          backgroundColor: 'rgba(75, 192, 192, 0.2)',
          fill: false,
          tension: 0.1
        }]
      },
      options: {
        plugins: {
          legend: {
            display: false // Hide the legend
          },
          title: {
            display: true,
            text: '지난 90일 가격 추이' // Chart title
          }
        },
        scales: {
          x: {
            type: 'time',
            time: {
              unit: 'day'
            }
          },
          y: {
            min: yMin,
            title: {
              display: true,
              text: 'Price'
            }
          }
        }
      }
    });
  });
</script>
```

---

## Static

사용한 css, js 코드에 대한 설명도 추가하려고 했으나, 기본적인 수준이고 내용이 크게 중요하지 않아 생략한다.

---

#### 수정 필요 사항

1. 404 에러 등 에러 처리
2. 제품 과거 최고 가격 대비 할인율 내림차순 정렬
3. 파트너스 공지 추가
4. 온라인으로 접속했을 때 너무 느림.(자바스크립트가 반영되기 전 값들이 눈에 보임.) 더 효율적인 방법 찾기.
5. 모바일 접근성 확보
6. 필터 최적화 및 추가

#### 같은 주제 게시글 모음

{{< related_posts tag="coupang tv" >}}
