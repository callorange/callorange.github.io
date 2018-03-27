---
title: Django setting 관리하기(2) - 비밀키 외부파일로 관리
keyword: django, settings, Execution environment, SECRET_KEY, API KEY, configparser, ini, json
description: 
date: 2018-03-26 21:33
categories:
- django
tags:
- Django setting 관리하기
---
# `Django settings` 비밀키 설정 외부 파일에서 로드하기
`SECRET_KEY`나 API연동을 위한 ACCESS KEY는 github같이 개방된 저장소에 올리면 망할 수 있으니 별도 폴더로 빼놓고 동적으로 로딩시키는 법을 정리해 보자

1. 환경변수
    - 가장 단순한 방법. 세팅할 값이 많아질수록 관리가 안되니 패스.
2. 외부파일 로드
    1. ini파일 - configparser
    2. json파일 - json
    3. python파일

---

# 환경변수로 지정하기
가장 단순한 방법.  
세팅해야 할 값이 많아 질 수록 환경변수 세팅이 관리가 안된다.

```python
SECRET_KEY = os.environ.get('SECRET_KEY', None)
```

---

# 외부파일 로드
별도 경로에 파일을 놓고 해당 파일들을 로딩하는 방법이다.  
`.gitignore`에 해당 폴더를 추가하고 같은 폴더에 넣어 놓으면 이후 배포시에 해당 폴더만 주의해서 배포해 주면 된다.

1. 베이스 디렉토리와 같은 위치에 `.scecret` 디렉토리 생성
2. `.scecret` 디렉토리는 `.gitignore`로 해당 디렉토리는 버전관리에서 제외
3. 세팅은 local.py에 한다고 가정.
    - 각 파일마다 세팅법은 동일하니까.

>project dir: django-practice  
>base dir: django-practice/myproject  
>secret dir: django-practice/.secret  
>git dir: django-practice/.git  

```shell
(django-practice)  callorange > ~/projects/practice/django-practice >  master>
> tree . -a
.
├── .git
│   └── ...
├── .gitignore
├── .python-version
├── .secret
└── myproject
    ├── config
    │   ├── __init__.py
    │   ├── settings
    │   │   ├── __init__.py
    │   │   ├── base.py
    │   │   ├── dev.py
    │   │   ├── local.py
    │   │   └── production.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── db.sqlite3
    └── manage.py
```

---

## 외부 파일을 불러오기 위한 공통설정
`base.py`에 외부 파일을 불러오기 위한 공통 설정을 해보자.  
안해도 되지만 굳이 여러번 쓰기는 귀찮으니까 한번만 설정해 놓자

```python
# base.py
ROOT_DIR = os.path.dirname(BASE_DIR)
SECRET_DIR = os.path.join(ROOT_DIR, '.secret') # .secret 디렉토리 경로를 얻어왔다.
```

## ini파일 - configparser
`.secret` 폴더에 local.ini을 만들고 configparser로 로딩한다.

```ini
# .secret/loca.ini
[DEFAULT]
SECRET_KEY = ASDF!@KJWLQKEF
DEBUG = True
```

```python
# local.py
import configparser
...
# configparser로 ini 파일 로딩
config = configparser.ConfigParser()
config.read(os.path.join(SECRET_DIR, 'local.ini'))
...
# 값 세팅
SECRET_KEY = config["DEFAULT"]["SECRET_KEY"] #
DEBUG = config["DEFAULT"]["DEBUG"]
```

ini 파일의 섹션명을 꼭 넣어줘야 한다.  
DATABASES같이 딕셔너리 형태로 관리되는 파일은 ini로 불러오기가 영 귀찮아 진다.

## json파일 - json
`.secret` 폴더에 local.json을 만들고 json파일을 읽어 로딩한다.

```json
# .secret/loca.json
{
  "SECRET_KEY": "...",
  "DEBUG": true,
  "DATABASES": {
      "default": {
          "ENGINE": "django.db.backends.sqlite3",
          "NAME": "os.path.join(BASE_DIR, 'db.sqlite3')" # 파이썬 코드같은 경우 일단 텍스트로 넣는다
      }
  }
}
```

```python
# local.py
import json
...
# json파일을 일어와서 json으로 로딩. dict객체로 온다.
with open(os.path.join(SECRET_DIR, 'local.json')) as f:
    secret_json_local = json.loads(f.read())
...
SECRET_KEY = secret_json_local["SECRET_KEY"]
DEBUG = secret_json_local["DEBUG"]
# os.path처럼 파이썬 코드가 들어있던 부분은 eval() 함수로 처리해 줘야 한다.
secret_json_local["DATABASES"]["default"]["NAME"] = eval(secret_json_local["DATABASES"]["default"]["NAME"])
DATABASES = secret_json_local["DATABASES"]
```

1. 장점
    - `DATABASES`처럼 dict 형태로 넣어줘야 하는 경우에도 편하게 넣을 수 있다.  
2. 단점
    - 상대 경로를 쓰는 경우처럼 파이썬 코드로 값을 세팅하는 경우(ex: sentry에서 git경로 세팅)에는 `eval()` 함수로 실제 실행한 결과를 처리해 줘야 한다.
    - 특정 패키지가 필요한 경우(ex: sentry는 raven을 import해야 한다)에는 별도로 import 해줘야 한다.

일일이 `eval()`처리를 하기 보다 자동으로 순회하면서 eval() 시킨 결과만 바로 할당해 보자

```python
# json으로 로딩한 dict객체 안에서 eval()을 모두 처리 한뒤에 dict객체를 현재 세팅에 할당하는 함수
def set_config(obj, module_name=sys.modules[__name__]):
    # dict나 list에서 for문으로 순회하면서 다시 객체 타입 체크.
    def parse_dict(obj):
        # str이면 eval(). dict나 list면 재귀호출.
        def is_str(obj, key, value):
            if isinstance(value, str):
                try:
                    obj[key] = eval(value)
                except:
                    pass
            else:
                parse_dict(value)
        # list랑 dict 구분
        if isinstance(obj, list):
            for num, value in enumerate(obj):
                is_str(obj, num, value)
        elif isinstance(obj, dict):
            for key, value in obj.items():
                is_str(obj, key, value)

    parse_dict(obj)
    # eval 처리를 완료한 값을 setattr로 바로 할당한다.
    # 기존 값을 교체해버리는것에 유의.
    for key, value in obj.items():
        setattr(module_name, key, value)

# 함수 위치가 같은 파이썬 파일일때
set_config(secret_json_local)
# 함수 위치가 base.py같이 다른 파일에 있을때
set_config(secret_json_local, sys.modules[__name__])
```
