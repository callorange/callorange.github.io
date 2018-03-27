---
title: Django setting 관리하기(1) - 실행환경별 세팅 파일 관리
keyword: django, settings, local, dev, production, Execution environment
description: 
date: 2018-03-26 20:07
categories:
- django
tags:
- Django setting 관리하기
---
# Django settings 실행환경별 관리
실행환경 별로 settings 파일을 나눠보자.

이전글인 [Django가 로딩 및 Response 하는 순서 - 1.3 settings.py 로딩](/django/2018/03/13/install4/){:target='_blank'}에 설명 해놨던 내용을 참조하자.  
__각 실행환경에서 환경변수에 로딩할 파일명만 제대로 지정 된다면 만사 OK!__

계획을 짜보자.  
1. 기본적인 공통 세팅을 가지고 있는 파일
2. 실제 환경변수에 따라 실행환경에 맞는 세팅을 가지고 있는 파일
  - 실행환경에 따라 여러개 일 수 있다. 

---

# settings.py 패키지화
굳이 패키지로 만들지 않아도 되지만 패키지로 만들고 그안에 파이썬 파일을 몰아 놓는게. 훨씬 보기 좋으므로 진행하자.

1. setting관련 패키지를 만들자  
    1. 수동
        - `settings` 폴더 생성 -> `__init__.py` 생성
    2. pyCharm
        - `settings.py` 우클릭 -> `refactor` -> `Convert to Python Package`
2. 기존내용을 새로 사용할 공통 세팅용 파일로 옮기기
  - 여기선 `base.py`로 생성
  - 파일명은 굳이 정해지지 않았으니 알아보기 편한 이름으로 하자
3. `base.py`에서 `BASE_DIR` 수정
  - 패키지화 하면서 기존보다 디렉토리 path가 한단계 더 들어왔기 때문에 `os.path.dirname()`을 한번 더 써줘야 base dir이 제대로 지정된다.

```shell
(django-practice)  callorange > ~/projects/practice/django-practice>
> tree .
.
└── myproject
    ├── config
    │   ├── __init__.py
    │   ├── settings
    │   │   ├── __init__.py
    │   │   └── base.py # 이제 이 파일을 기본 공통 세팅을 담는 파일로 사용할 거다.
    │   ├── urls.py
    │   └── wsgi.py
    ├── db.sqlite3
    └── manage.py

3 directories, 7 files
```

---

# 실행 환경별 파일 생성하기
이제 각 실행 환경별 세팅을 만들 차례다.

1. `settings` 패키지에 실행 환경별로 파일을 만든다.
  - 여기선 각각 `local.py`, `dev.py`, `production.py`로 생성
2. 만들어진 파일에서 `base.py`을 import 시킨다.
  ```python
  from .base import * # import *로 base 모듈의 모든 변수를 로딩. 헷갈리지 말것.
  ```
3. 실행환경 별로 달라질 세팅을 각 파일로 옮긴다.
  - `ALLOWED_HOSTS`, `DATABASES`, `DEBUG` 등
  - 변수명이 같다면 `base.py`에 있던 세팅을 덮어 쓴다. dict 변수 같은 경우엔 조심하자.

```shell
(django-practice)  callorange > ~/projects/practice/django-practice>
> tree .
.
└── myproject
    ├── config
    │   ├── __init__.py
    │   ├── settings
    │   │   ├── __init__.py
    │   │   ├── base.py
    │   │   ├── dev.py # 개발서버용
    │   │   ├── local.py # 로컬테스트용
    │   │   └── production.py # 실제 최종 배포용
    │   ├── urls.py
    │   └── wsgi.py
    ├── db.sqlite3
    └── manage.py

3 directories, 10 files
```

---

# 실행시 세팅파일 지정

## `manage.py`, `wsgi.py`에서 기본값 변경(선택)
아래와 같이 수정하면 기본적으로 `config.settings.local`로 로딩하게 된다.  
별도로 환경변수가 지정되지 않았다면 기본값으로 지정한 세팅으로 로딩한다.

```python
# manage.py
# os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.local")

# wsgi.py
# os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.local")
```

## 환경변수 세팅
운영체제의 `DJANGO_SETTINGS_MODULE` 환경변수를 원하는 값으로 세팅한다.
각 운영체제별 환경변수 세팅법은 구글 검색으로...(환경변수 세팅까지 설명하자면 글이 너무 길어진다)

## runserver시 파라미터 지정(개발도중 필요할 때만)
runserver 자체는 개발용 로컬서버로만 사용하니 개발 및 운영 서버에서 사용할 내용은 아니다.
```shell
# manage.py runserver --settings=<패키지.모듈>
./manage.py runserver --settings=config.settings.local
```


