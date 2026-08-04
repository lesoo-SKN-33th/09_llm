# PyCharm에서 OpenAI API 키 설정하기

## 1. Conda-forge 가상환경과 PyCharm 설정하기

수업 실습은 다른 프로젝트의 패키지와 충돌하지 않도록 `llm_env` 가상환경에서 진행한다. Python 3.12와 `pip`를 Conda-forge 채널에서 설치한다.

### macOS

Terminal 또는 PyCharm의 Terminal에서 다음 명령을 실행한다.

```bash
conda create -n llm_env -c conda-forge python=3.12 pip
conda activate llm_env
```

`conda activate` 명령이 동작하지 않으면 다음 명령을 실행한 뒤 Terminal을 다시 연다.

```bash
conda init zsh
```

### Windows

Miniforge Prompt, Anaconda Prompt 또는 PyCharm의 Terminal에서 다음 명령을 실행한다.

```powershell
conda create -n llm_env -c conda-forge python=3.12 pip
conda activate llm_env
```

PowerShell에서 `conda activate` 명령이 동작하지 않으면 다음 명령을 실행한 뒤 PowerShell을 다시 연다.

```powershell
conda init powershell
```

### PyCharm 프로젝트와 인터프리터 연결

1. PyCharm에서 `08_llm` 폴더를 프로젝트 최상위로 연다.
2. PyCharm 오른쪽 하단에 `<인터프리터 없음>`을 클릭한다.
3. `Project → Python Interpreter → Add Interpreter → Add Local Interpreter`를 선택한다.
4. `Conda Environment → Existing environment`에서 `llm_env`의 Python을 선택한다.
5. PyCharm Terminal을 새로 열고 다음 명령으로 환경을 확인한다.

```bash
conda env list
python --version
```

`conda env list`에서 `llm_env` 왼쪽에 `*`가 표시되고 `python --version`이 `3.12.x`이면 준비가 완료된 것이다. 프로젝트 탐색기에서 `.idea`, `.gitignore`, `requirements.txt`가 같은 수준에 보이는지도 확인한다.

## 2. 패키지 설치하기

PyCharm Terminal에서 `llm_env`가 활성화되었는지 확인한 뒤 `requirements.txt`에 정리된 수업 공통 패키지를 한 번에 설치한다. 

```bash
python -m pip install -r requirements.txt
```

`requirements.txt`에는 다음 패키지 범주가 포함되어 있다.

- 노트북 실행: `jupyterlab`, `ipykernel`, `ipywidgets`
- 데이터 처리와 시각화: `numpy`, `pandas`, `matplotlib`, `scikit-learn`, `Pillow`
- HTTP 요청과 진행률 표시: `requests`, `tqdm`
- OpenAI API와 음성 입출력: `openai[realtime]`, `python-dotenv`, `sounddevice`

설치가 끝나면 `llm_env`를 Jupyter 커널로 등록한다.

```bash
python -m ipykernel install --user --name llm_env --display-name "Python (llm_env)"
```

PyCharm에서 노트북을 열고 커널 목록에서 `Python (llm_env)`를 선택한다.

## 3. `.env` 파일 만들기

1. PyCharm에서 `08_llm` 폴더를 마우스 오른쪽 버튼으로 클릭한다.
2. `New → File`을 선택한다.
3. 파일 이름을 `.env`로 입력한다.
4. 다음 형식으로 본인이 발급받은 키를 저장한다.

```dotenv
OPENAI_API_KEY=본인이_발급받은_API_키
```

따옴표는 필수가 아니다. `OPENAI_API_KEY` 앞뒤에 공백을 넣지 않으며, 키 값을 코드셀이나 출력에 복사하지 않는다. 강사의 키를 학생에게 배포하지 않고 학생마다 자신의 OpenAI 프로젝트 키를 사용한다.

`08_llm/.gitignore`에는 `.env`를 제외하는 규칙이 포함되어 있다. 공유할 때는 값이 비어 있는 `.env.example`만 사용한다.

## 4. 노트북에서 불러오기

```python
import os

from dotenv import find_dotenv, load_dotenv
from openai import OpenAI

dotenv_path = find_dotenv(usecwd=True)
if not dotenv_path:
    raise FileNotFoundError(
        "08_llm 프로젝트 최상위에 .env 파일을 만든 뒤 현재 셀을 다시 실행한다."
    )

load_dotenv(dotenv_path, override=False)

if not os.getenv("OPENAI_API_KEY"):
    raise RuntimeError(".env 파일의 OPENAI_API_KEY 값을 확인한다.")

client = OpenAI()
print("OpenAI 클라이언트 준비 완료")
```

`find_dotenv(usecwd=True)`는 현재 Jupyter 작업 폴더부터 상위 폴더로 이동하며 `.env`를 찾는다. `load_dotenv()`는 파일의 값을 현재 Jupyter 커널의 환경 변수에 등록한다. OpenAI SDK의 `OpenAI()`는 `OPENAI_API_KEY`를 자동으로 읽으므로 키를 생성자에 직접 전달하지 않는다.

## 5. 문제 해결하기

- `conda` 명령을 찾을 수 없으면 Miniforge가 설치되어 있는지 확인하고 macOS는 `conda init zsh`, Windows PowerShell은 `conda init powershell`을 실행한 뒤 Terminal을 다시 연다.
- PyCharm Terminal에 `(llm_env)`가 표시되지 않으면 `conda activate llm_env`를 실행하고 Python Interpreter도 `llm_env`로 다시 선택한다.
- `.env 파일을 찾을 수 없다`는 오류가 나오면 PyCharm에서 `08_llm`을 프로젝트로 열었는지 확인한다.
- `OPENAI_API_KEY`가 없다는 오류가 나오면 파일 이름이 정확히 `.env`인지, 변수 이름과 값 사이에 오타가 없는지 확인한다.
- 패키지를 찾을 수 없다는 오류가 나오면 노트북 커널과 PyCharm Terminal이 같은 Python Interpreter를 사용하는지 확인한다.
- `.env`를 수정한 뒤 기존 값이 유지되면 Jupyter 커널을 재시작하고 인증 셀부터 다시 실행한다.
- 키가 노출되었다면 해당 키를 즉시 폐기하고 새 키를 발급한다.

운영 서버에서는 `.env` 파일 대신 배포 플랫폼의 Secrets 또는 클라우드 Secret Manager를 사용한다.

## 공식 참고 자료

- [OpenAI API Quickstart](https://developers.openai.com/api/docs/quickstart)
- [OpenAI Production best practices](https://developers.openai.com/api/docs/guides/production-best-practices)
- [python-dotenv](https://pypi.org/project/python-dotenv/)
