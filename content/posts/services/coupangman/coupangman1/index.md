+++
title = 'coupang TV[0] - 프로젝트 구조 설계'
date = 2024-08-07T05:31:40+09:00
categories = ["services"]
tags = ["indie hacker", "coupang tv"]
+++

이 프로젝트는 쿠팡에서 판매하는 TV 정보를 수집하여 리스트를 제공한다. 매일 업데이트하여 쿠팡에서는 제공하지 않는 과거 가격과 정리된 TV 제품 스펙을 제공한다. 또한 다양한 필터를 활용하여 소비자는 원하는 TV를 추릴 수 있다.

<!--more-->

쿠팡에서 TV 리스트를 수집하고 다시 제공할 때, 쿠팡 파트너스 링크를 활용하여 수수료 수익을 창출한다.

## 프로젝트 레이아웃

```code
CoupangTV
 ├─app
 │  ├─models
 │  │  ├─__init__.py
 │  │  ├─models.py
 │  │  └─update_or_create_product.py
 │  ├─routes
 │  │  ├─__init__.py
 │  │  └─routes.py
 │  ├─scrapers
 │  │  ├─__init_.py
 │  │  ├─scraper_basic.py
 │  │  ├─scraper_feature.py
 │  │  ├─scraper_url2.py
 │  │  └─utils.py
 │  ├─static
 │  │  ├─css
 │  │  │  └─style.css
 │  │  ├─images
 │  │  └─js
 │  │     └─scripts.js
 │  ├─templates
 │  ├─__init__.py
 │  ├─celery.py
 │  ├─config.py
 │  └─tasks.py
 ├─instantce
 │  └─yourdatabase.db
 ├─celery_worker.py
 ├─run.py
 └─requrirements.txt
```

## 프로젝트 구현

이 풀스택 프로젝트는 총 3가지 부분을 개발해야한다.

1. 백엔드(스크래핑)
2. 프론트엔드
3. 서버

우선 스크래핑 기능을 개발하고, 스크래핑된 데이터를 바탕으로 웹을 로컬 환경에서 구축한다. 그리고 그걸 서버에 배포한다.
이 프로젝트는 flask 기반으로 만들어졌기 때문에 백엔드, 프론트 엔드는 파이썬을 기반으로 구축한다.

서버는 aws 등 클라우드 컴퓨팅 서비스를 활용할 수 있겠지만, 고정비용을 절약하고 서버 의존도를 줄이기 위해 프로젝트 초반에는 직접 안쓰는 노트북을 활용해서 서버를 구축한다.

#### 같은 주제 게시글 모음

{{< related_posts tag="coupang tv" >}}
