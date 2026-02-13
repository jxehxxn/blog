---
layout: post
title: "OpenStack Zero-to-Hero: Supplement - Debugging OpenStack Code (PDB & Logs)"
---

구글링해도 답이 안 나오는 에러.
로그에는 `Internal Server Error (500)` 한 줄뿐.
이럴 땐 **코드를 까야 합니다.**

오픈스택은 **Python**으로 짜여 있습니다. 소스 코드가 서버에 그대로 있습니다.
직접 수정하고 디버깅하는 **"Developer Level"**의 트러블슈팅을 배웁니다.

---

## 1. Locating the Code

Kolla-Ansible로 설치했다면 코드는 컨테이너 안에 있습니다.

```bash
docker exec -it nova_api bash
cd /var/lib/kolla/venv/lib/python3.10/site-packages/nova/
```
여기가 보물창고입니다.

---

## 2. Reading the Traceback

로그 파일(`/var/log/kolla/nova/nova-api.log`)에 있는 파이썬 에러 스택 트레이스(Stack Trace)를 무서워하지 마세요.
**가장 마지막 줄**이 범인입니다.

`File "/.../nova/compute/api.py", line 1234, in create`
`KeyError: 'image_id'`

-> `nova/compute/api.py` 파일의 1234번째 줄에서 `image_id`라는 키를 찾지 못했다는 뜻입니다.

---

## 3. Remote Debugging (`pdb`)

"도대체 저 변수에 뭐가 들어있길래 에러가 나지?"
코드를 열고 중단점(Breakpoint)을 심습니다.

```python
# nova/compute/api.py 수정
def create(...):
    import pdb; pdb.set_trace() # 중단점 추가
    ...
```

하지만 컨테이너 환경에서는 `pdb` 쉘을 띄우기 어렵습니다.
대신 **`remote_pdb`**를 씁니다.

1.  `pip install remote-pdb`
2.  코드 삽입:
    ```python
    from remote_pdb import RemotePdb
    RemotePdb('0.0.0.0', 4444).set_trace()
    ```
3.  컨테이너 재시작.
4.  API 요청을 날리면 프로세스가 멈춥니다.
5.  `telnet <Container_IP> 4444`로 접속하면 디버깅 쉘이 뜹니다!

---

## 4. Hotfixing (Monkey Patching)

버그를 찾았습니다. 공식 패치가 나올 때까지 기다릴 수 없습니다.
직접 코드를 고쳐서 돌려야 합니다.

1.  컨테이너 안의 `.py` 파일을 수정합니다. (`vi`가 없으면 `apt install vim` 하거나 밖에서 `docker cp`로 넣습니다)
2.  `__pycache__` 폴더 삭제 (혹시 모르니까).
3.  컨테이너 재시작 (`docker restart nova_api`).

**주의:** 컨테이너를 재생성(`kolla-ansible deploy`)하면 수정 사항이 날아갑니다.
영구적으로 적용하려면:
*   수정된 파일을 호스트의 특정 경로에 둡니다.
*   `globals.yml`의 `extra_volumes` 설정을 이용해 그 파일을 컨테이너의 원래 파일 위치에 **마운트(Bind Mount)**합니다.

---

## 5. Summary

오픈스택 엔지니어의 레벨은 여기서 갈립니다.
*   **Lv 1:** 로그 보고 구글링한다.
*   **Lv 2:** 설정 파일을 수정한다.
*   **Lv 3 (Guru):** **파이썬 코드를 열고 로직을 따라간다.**

소스 코드는 최고의 문서입니다. 두려워 말고 열어보세요.
