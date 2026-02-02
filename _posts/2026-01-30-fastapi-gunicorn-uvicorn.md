---
layout: post
title: "FastAPI / gunicorn / uvicorn / 동기 / 비동기 "
---

##### [FastAPI](https://fastapi.tiangolo.com/ko/)
FastAPI에서 작성된 코드는 단독으로 실행되어서 Web API로써 동작할 수 없다.  
HTTP 요청을 받아 FastAPI 인스턴스에게 전달하고 인스턴스로부터 데이터를 받아 HTTP 응답을 제공하는 서버의 역할을 하는 것은 Uvicorn이다.

1. async def와 def의 차이
   - FastAPI 공식문서는 비동기 처리 라이브러리일 경우 async/await를 사용하고 그렇지 않으면 def를 사용 권장
   - def의 경우 외부 쓰레드 풀에서 다이렉트로 실행하여 서버가 블록킹되지 않음
   - async def는 외부 쓰레드 풀에서 실행하지 않아 블록킹됨

##### [Uvicorn](https://leapcell.io/blog/ko/fastapi-uvicorn-beulleijeing-seupideu-the-tech-behind-the-hype)
> uvloop와 httptools를 기반으로 구축된, 비동기 처리에 최적화된 Python ASGI(Asynchronous Server Gateway Interface) 서버로 FASTApi와 함께 사용

- 특징   
    Python의 asyncio를 기반으로 작동하며, uvloop를 사용해 기존 asyncio 이벤트 루프보다 빠른 처리 속도를 제공
- 용도   
    FASTApi나 Django와 같은 비동기 웹 프레임워크를 실행하는 핵심 서버 구현체로 사용
- 장점    
    비동기 웹 어플리케이션 환경에서 최상의 성능을 제공
- 배포    
    `개발 환경에서 단독으로 사용하기 좋으며, 운영 환경에서는 안정성을 위해 Gunicorn과 결합하여 Uvicorn 워커를 사용하는 것을 권장`

##### [Gunicorn](https://gunicorn.org/)
> Python WSGI(Web Server Gateway Interface) HTTP 서버로 FastAPI 와 같은 웹 프레임워크와 Nginx 등의 `웹 서버 사이`에서 요청을 처리하는 경량 WAS

1. 특징 및 장점  
   - 높은 호화성 - WSGI 규격을 따르는 대부분의 파이썬 프레임워크와 호환  
   - 프리포크 모델 - 요청이 들어오기 전에 미리 자식 프로세스(worker)를 만들어두어, 빠른 응답 속도 제공
   - 설정 간소화 - 복잡한 설정 없이 간편하게 실행가능
   - 성능 - 가볍고 빠르며, 안정적인 리소스 관리
2. 작동방식  
   - 일반적으로 Nginx(리버스 프록시) 뒤에서 작동하며, Nginx가 정적 파일을 처리하고 Gunicorn이 동적 파이썬 어플리케이션 요청을 처리하는 구조로 사용 

##### 왜 Gunicorn과 Uvicorn을 함께 사용하는가?
> Gunicorn은 worker 관리 및 로드 밸런싱을 담당하고 Uvicorn은 비동기 요청 처리를 담당하여 안정적이고 효육적인 서버를 운영

1. Gunicorn (프로세스 관리):
   - 멀티 프로세싱: 여러 개의 워커(worker) 프로세스를 띄워 멀티 코어 CPU를 최대한 활용
   - 안정성 및 복구: 워커 프로세스가 예기치 않게 종료되면 Gunicorn이 감지하고 자동으로 재시작하여 서버의 가용성 유지
   - 유연한 관리: 서버 가동 중에도 다운타임 없이 워커 수를 조정하거나 설정을 변경하는 등 관리가 용이함
2. Uvicorn (고성능 비동기 처리):   
   - ASGI 지원: FastAPI 같은 ASGI 표준을 따르는 비동기 프레임워크를 지원하는 초고속 서버
   - 비동기 최적화: uvloop와 httptools를 기반으로 하여 비동기 요청을 매우 빠르게 처리함

##### 이상적인(권장) 설정 (workers + threads)
> - Worker 수 : (2 * CPU 코어수) + 1  
> - Thread 수 : worker 클래스를 Uvivorn으로 설정 했다면, Thread는 1로 두거나 설정하지 않음

Uvicorn은 자체적으로 Gunicorn을 지원하는 Worker class(uvicorn.workers.UvicornWorker)를 포함하여   
Gunicorn 실행시 이 클래스 경로를 Gunicorn 실행시 -k 파라미터로 전달하여 동작  

<pre><code>gunicorn -w 9 -k uvicorn.workers.UvicornWorker --threads 1 main:app
gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:80
</code></pre>

- 설정값 조정
  - I/O 바운드 작업이 많은 경우: 권장 사항 적용
  - CPU 바운드 작업이 많은 경우: 워커 수를 CPU 코어 수와 같게 줄이거나 조금 더 많게 설정하는 것이 효율적일 수 있음 

##### 항상 Gunicorn과 Uvicorn을 같이 사용해야 하는가?
> 아니다. 

- 베어메탈 서버(Bare Metal Server)일 경우는 Uvicorn + Gunicorn을 같이 활용
- 분산 시스템 (Kubernetes, Docker Swarm)일 경우는 Uvicorn만 사용하여 클러스터 수준에서 수행

##### Uvicorn, FastAPI 비동기 매커니즘

파이썬으로 구현된 ASGI Server, ASGI Framework는 프로세스 기반으로 구현

- WSGI  
WSGI는 요청을 받고 응답해주는 역할을 합니다. 웹 서버(Nginx)와 웹 프레임워크(Flask) 사이에 존재합니다. Tomcat과 비슷합니다.  
WSGI는 웹 프레임워크인 Flask에서 작성된 function을 호출하는 역할을 합니다.  
비동기 처리가 어렵습니다.

- ASGI
ASGI는 요청에 대해 비동기적으로 처리할 수 있습니다.

프레임워크가 비동기 형태로 제공하더라도 라이브러리가 비동기 형태로 제공하지 않는다면 비동기 프레임워크를 쓰는게 큰 의미가 없습니다.

- 라이브러리가 비동기인지 확인하는 방법
  1. inspect 모듈 사용  
    <pre><code>import inspect
  import asyncio  
  async def my_async_func():
    await asyncio.sleep(1)
  
  def my_sync_func():
    pass
  
  print(inspect.iscoroutinefunction(my_async_func))  # True
  print(inspect.iscoroutinefunction(my_sync_func))  # False
  </code></pre>
  
  2. 문서 및 소스 코드 확인
     - 문서 확인: 공식 문서 확인 (async/await 사용 확인)
     - async def 정의: 함수나 메소드가 async def를 정의되어 있다면 비동기
     - await 키워드 여부: 함수 호출 시 결과를 얻기 위해 await를 사용해야 한다면 비동기
     
  3. asyncio.iscoroutinefunction() 사용
  
##### FastAPI에서 비동기를 사용하는 이유와 경우

> DB 쿼리, 외부 API 요청, 파일 I/O 등 I/O Bound 작업이 많아 대기 시간이 발생할 때 성능을 극대화하기 위해 사용

1. FastAPI에서 비동기를 사용하는 주요 이유
   - 높은 동시성 확보:
   - I/O 바운드 작업의 최적화:
   - ASGI 기반:
   - 비동기식 코드의 가독성: async / await 구문은 복잡한 비동기 로직을 동기 코드처럼 명확하게 작성할 수 있게하여 가독성을 향상
   - 효율적인 리소스 사용: 
   
2. FastAPI에서 비동기를 사용하는 주요 상황
   - 외부 API 호출 및 웹 크롤링: 비동기를 지원하는 라이브러리를 사용하여 다른 서버의 응답을 기다리는 동안 블로킹을 방지할 때
   - 비동기 데이터베이스 작업: 비동기 ORM/드라이버를 사용하여 DB I/O를 처리할 때
   - 파일 I/O 및 데이터 처리: 대용량 파일 읽기/쓰기나 네트워크 소켓 통신을 수행할 때
   - 높은 동시성 필요: 짧은 시간에 많은 요청이 들어오는 채팅, 실시간 스트리밍, 대규모 마이크로서비스 환경

3. 주의 사항
   - CPU Bound 작업: 영상 처리, 머신러닝 등 CPU를 많이 사용하는 작업은 비동기(async def)를 사용해도 멈치지 않으며,  
   오히려 동기 함수(def)를 사용하는 것이 스레드 풀에서 동작하여 더 유리할 수 있음
   - 블로킹 라이브러리: request, time.sleep() 등 동기식 라이브러리를 async def(비동기) 내에서 사용하면 전체 어플리케이션이 멈추므로,  
   반드시 비동기 라이브러리를 사용해야 한다.

<pre><code>from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.get("/")
async def read_data():
    await asyncio.sleep(1) 
    return {"message": "Async data"}
</code></pre>

##### FastAPI 비동기 라우트 함수 적용
async def를 사용하여 라우트 함수를 정의하면 입출력(I/O) 바운드 작업 (DB, 외부 API 호출)시 이벤트 루프를 차단하지 않아 높은 동시성을 확보할 수 있다.  
CPU 작업이 많은 경우 def를 사용하여 쓰레드풀에서 실행하는 것이 좋다.

- 비동기 라우트 정리
- async def 사용: 비동기 라이브러리(httpx, database)와 함께 사용하여 I/O 블로킹을 방지
- await 키워드: 비동기 함수가 결과를 반환할 때까지 기다리며, 그동안 다른 요청을 처리
- 적절한 상황: 데이터베이스, 파일 I/O, 외부 API 호출 등 네트워크 요청이 많은 경우 사용
- 주의사항: async def 안에서 time.sleep() 같은 동기적 블로킹 함수를 사용하면 서버 전체가 멈출 수 있으므로, 반드시 비동기 함수를 사용해야 한다.

---

https://velog.io/@bbkyoo/gunicorn-uvicorn  
https://yscho03.tistory.com/328
