遇到了pypi的依赖问题
这是 uv 的输出
```
uv pip install  -r requirements.txt
    Updated https://github.com/MeetAgent/storage-client-sdk.git (fd2a70510feb2e13124cdbf923ac6827dd820002)
  × No solution found when resolving dependencies:
  ╰─▶ Because only asset-storage-sdk==0.4.0 is available and asset-storage-sdk==0.4.0 depends on pydantic>=2.13.4,<3.0.0, we can conclude that all versions of asset-storage-sdk depend on
      pydantic>=2.13.4,<3.0.0.
      And because you require asset-storage-sdk and pydantic==2.11.0, we can conclude that your requirements are unsatisfiable.
```
这是pip的输出
```
 pip install  -r requirements.txt 
Collecting asset-storage-sdk @ git+https://github.com/MeetAgent/storage-client-sdk.git@0.4.0 (from -r requirements.txt (line 3))
  Cloning https://github.com/MeetAgent/storage-client-sdk.git (to revision 0.4.0) to /tmp/pip-install-4cndyhpk/asset-storage-sdk_5a3f05ae3eb3485fa6640cdc0b9e6a19
  Running command git clone --filter=blob:none --quiet https://github.com/MeetAgent/storage-client-sdk.git /tmp/pip-install-4cndyhpk/asset-storage-sdk_5a3f05ae3eb3485fa6640cdc0b9e6a19
  Running command git checkout -b 0.4.0 --track origin/0.4.0
  Switched to a new branch '0.4.0'
  branch '0.4.0' set up to track 'origin/0.4.0'.
  Resolved https://github.com/MeetAgent/storage-client-sdk.git to commit fd2a70510feb2e13124cdbf923ac6827dd820002
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Preparing metadata (pyproject.toml) ... done
Requirement already satisfied: aiomysql==0.3.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 1)) (0.3.0)
Requirement already satisfied: aio-pika==9.5.5 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 2)) (9.5.5)
Requirement already satisfied: annotated-types==0.7.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 4)) (0.7.0)
Requirement already satisfied: anyio==4.9.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 5)) (4.9.0)
Requirement already satisfied: asgi-correlation-id==4.3.4 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 6)) (4.3.4)
Requirement already satisfied: async-timeout==5.0.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 7)) (5.0.1)
Requirement already satisfied: asyncmy==0.2.10 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 8)) (0.2.10)
Requirement already satisfied: bcrypt==4.3.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 9)) (4.3.0)
Requirement already satisfied: beanie==2.0.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 10)) (2.0.1)
Requirement already satisfied: beautifulsoup4==4.12.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 11)) (4.12.0)
Requirement already satisfied: billiard==4.2.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 12)) (4.2.2)
Requirement already satisfied: cachetools==5.5.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 13)) (5.5.0)
Requirement already satisfied: certifi==2024.12.14 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 14)) (2024.12.14)
Requirement already satisfied: cffi==2.0.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 15)) (2.0.0)
Requirement already satisfied: chardet==5.2.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 16)) (5.2.0)
Requirement already satisfied: charset-normalizer==3.4.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 17)) (3.4.1)
Requirement already satisfied: click==8.1.8 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 18)) (8.1.8)
Requirement already satisfied: cryptography==46.0.7 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 19)) (46.0.7)
Requirement already satisfied: dnspython==2.7.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 20)) (2.7.0)
Requirement already satisfied: exceptiongroup==1.2.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 21)) (1.2.2)
Requirement already satisfied: fastapi==0.124.4 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 22)) (0.124.4)
Requirement already satisfied: fastapi-cli==0.0.7 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 23)) (0.0.7)
Requirement already satisfied: fastapi-pagination==0.12.34 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 24)) (0.12.34)
Requirement already satisfied: google-auth==2.37.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 25)) (2.37.0)
Requirement already satisfied: google-cloud-bigquery==3.28.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 26)) (3.28.0)
Requirement already satisfied: google-cloud-storage==2.19.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 27)) (2.19.0)
Requirement already satisfied: greenlet==3.1.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 28)) (3.1.1)
Requirement already satisfied: gunicorn==23.0.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 29)) (23.0.0)
Requirement already satisfied: h11==0.16.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 30)) (0.16.0)
Requirement already satisfied: httpcore==1.0.9 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 31)) (1.0.9)
Requirement already satisfied: httptools==0.7.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 32)) (0.7.1)
Requirement already satisfied: httpx==0.28.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 33)) (0.28.1)
Requirement already satisfied: idna==3.10 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 34)) (3.10)
Requirement already satisfied: itsdangerous==2.2.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 35)) (2.2.0)
Requirement already satisfied: Jinja2==3.1.6 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 36)) (3.1.6)
Requirement already satisfied: loguru==0.7.3 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 37)) (0.7.3)
Requirement already satisfied: markdown-it-py==4.0.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 38)) (4.0.0)
Requirement already satisfied: MarkupSafe==3.0.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 39)) (3.0.2)
Requirement already satisfied: mdurl==0.1.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 40)) (0.1.2)
Requirement already satisfied: motor==3.7.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 41)) (3.7.1)
Requirement already satisfied: openpyxl==3.1.5 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 42)) (3.1.5)
Requirement already satisfied: msgspec==0.19.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 43)) (0.19.0)
Requirement already satisfied: packaging==24.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 44)) (24.2)
Requirement already satisfied: passlib==1.7.4 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 45)) (1.7.4)
Requirement already satisfied: patool==1.12 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 46)) (1.12)
Requirement already satisfied: pip-autoremove==0.10.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 47)) (0.10.0)
Requirement already satisfied: protobuf==6.33.5 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 48)) (6.33.5)
Requirement already satisfied: pycparser==2.22 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 49)) (2.22)
Requirement already satisfied: pycryptodome==3.22.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 50)) (3.22.0)
Requirement already satisfied: pydantic==2.11.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 51)) (2.11.0)
Requirement already satisfied: pydantic-settings==2.8.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 52)) (2.8.1)
Requirement already satisfied: pydantic_core==2.33.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 53)) (2.33.0)
Requirement already satisfied: Pygments==2.20.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 54)) (2.20.0)
Requirement already satisfied: PyMySQL==1.1.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 55)) (1.1.1)
Requirement already satisfied: python-dotenv==1.2.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 56)) (1.2.2)
Requirement already satisfied: python-jose==3.4.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 57)) (3.4.0)
Requirement already satisfied: python-multipart==0.0.26 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 58)) (0.0.26)
Requirement already satisfied: python-statemachine==3.2.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 59)) (3.2.0)
Requirement already satisfied: PyYAML==6.0.3 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 60)) (6.0.3)
Requirement already satisfied: redis==5.2.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 61)) (5.2.1)
Requirement already satisfied: python-redis-lock==4.0.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 62)) (4.0.1)
Requirement already satisfied: rich==14.2.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 63)) (14.2.0)
Requirement already satisfied: rich-toolkit==0.15.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 64)) (0.15.1)
Requirement already satisfied: shellingham==1.5.4 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 65)) (1.5.4)
Requirement already satisfied: sniffio==1.3.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 66)) (1.3.1)
Requirement already satisfied: soupsieve==2.8 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 67)) (2.8)
Requirement already satisfied: SQLAlchemy==2.0.40 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 68)) (2.0.40)
Requirement already satisfied: sqlalchemy-crud-plus==1.13.3 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 69)) (1.13.3)
Requirement already satisfied: starlette==0.49.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 70)) (0.49.1)
Requirement already satisfied: tenacity==9.1.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 71)) (9.1.2)
Requirement already satisfied: typer==0.20.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 72)) (0.20.0)
Requirement already satisfied: typing-inspection==0.4.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 73)) (0.4.0)
Requirement already satisfied: typing_extensions==4.12.2 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 74)) (4.12.2)
Requirement already satisfied: urllib3==2.6.3 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 75)) (2.6.3)
Requirement already satisfied: uvicorn==0.34.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 76)) (0.34.0)
Requirement already satisfied: uvloop==0.22.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 77)) (0.22.1)
Requirement already satisfied: xlrd==2.0.1 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 78)) (2.0.1)
Requirement already satisfied: google-genai==1.51.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 79)) (1.51.0)
Requirement already satisfied: mistune==3.1.0 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 80)) (3.1.0)
Requirement already satisfied: pytest==9.0.3 in ./.venv/lib/python3.12/site-packages (from -r requirements.txt (line 81)) (9.0.3)
INFO: pip is looking at multiple versions of asset-storage-sdk to determine which version is compatible with other requirements. This could take a while.
ERROR: Cannot install -r requirements.txt (line 3) and pydantic==2.11.0 because these package versions have conflicting dependencies.

The conflict is caused by:
    The user requested pydantic==2.11.0
    asset-storage-sdk 0.4.0 depends on pydantic<3.0.0 and >=2.13.4

Additionally, some packages in these conflicts have no matching distributions available for your environment:
    pydantic

To fix this you could try to:
1. loosen the range of package versions you've specified
2. remove package versions to allow pip to attempt to solve the dependency conflict

ERROR: ResolutionImpossible: for help visit https://pip.pypa.io/en/latest/topics/dependency-resolution/#dealing-with-dependency-conflicts

```

可以看出uv的输出更简短.