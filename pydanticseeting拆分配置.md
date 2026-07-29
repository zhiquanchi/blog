# Pydantic Settings 用法与配置拆分实践

## 一、什么是 Pydantic Settings

[Pydantic Settings](https://pydantic.dev/docs/validation/latest/concepts/pydantic_settings/) 是 Pydantic 生态中的配置管理库，它让你可以用类型安全的方式从环境变量、`.env` 文件、密钥文件、TOML/YAML/JSON 配置文件甚至命令行参数中加载应用配置。

核心特性：

- **类型安全**：基于 Pydantic 模型，配置字段天然带有类型校验
- **多源加载**：支持环境变量、dotenv 文件、secrets、配置文件、CLI 参数等多种来源
- **优先级机制**：不同来源有明确的优先级，高优先级覆盖低优先级
- **易于测试**：可以在初始化时直接传参覆盖配置，方便单元测试

## 二、安装与基础用法

### 安装

```bash
pip install pydantic-settings
```

### 最简示例

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    app_name: str = "my_app"
    debug: bool = False
    port: int = 8000

settings = Settings()
print(settings.app_name)  # 从环境变量 APP_NAME 读取，或使用默认值
```

`BaseSettings` 的初始化器会自动从环境变量中读取未显式传入的字段值。环境变量名默认与字段名一致（不区分大小写），可以通过 `env_prefix` 统一加前缀。

### 字段值优先级

从高到低依次为：

1. CLI 参数（如果启用了 `cli_parse_args`）
2. 初始化参数（`Settings(foo=...)`）
3. 环境变量
4. dotenv（`.env`）文件
5. secrets 目录
6. 字段默认值

### 环境变量命名

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix='MY_APP_')

    auth_key: str = "xxx"  # 读取 MY_APP_AUTH_KEY
    redis_host: str = "localhost"  # 读取 MY_APP_REDIS_HOST
```

可以通过 `validation_alias` 或 `alias` 为单个字段指定环境变量名，使用 `AliasChoices` 支持多个别名。

## 三、为什么要拆分配置

当应用规模增长，把所有配置都塞在一个 `Settings` 类里会导致：

- 类过于臃肿，字段成百上千
- 不同模块的配置耦合在一起，难以维护
- 无法按模块独立加载和验证
- 团队协作时容易产生冲突

Pydantic Settings 提供了多种方式来拆分配置，下面逐一介绍。

## 四、拆分配置的几种方式

### 方式一：嵌套模型（Nested Model）

最常见的拆分方式，按功能模块把相关字段聚合成子模型。

```python
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict

class DatabaseSettings(BaseModel):
    host: str = "localhost"
    port: int = 5432
    user: str
    password: str

class RedisSettings(BaseModel):
    host: str = "localhost"
    port: int = 6379
    db: int = 0

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_nested_delimiter='__')

    app_name: str = "my_app"
    database: DatabaseSettings
    redis: RedisSettings

settings = Settings()
print(settings.database.host)  # 从 DATABASE__HOST 读取
print(settings.redis.port)     # 从 REDIS__PORT 读取
```

**关键点**：

- 子模型继承自 `pydantic.BaseModel`，而不是 `BaseSettings`
- 通过 `env_nested_delimiter='__'` 指定嵌套分隔符，环境变量格式为 `DATABASE__HOST`
- 也可以用 JSON 格式一次性传入整个子模型：`export DATABASE='{"host":"localhost","port":5432}'`

**限制嵌套深度**：如果字段名本身含有下划线（如 `api_key`），可以用 `env_nested_max_split` 限制拆分深度：

```python
class LLMConfig(BaseModel):
    provider: str = "openai"
    api_key: str
    api_version: str = "2024-03-15"

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix='GENERATION_',
        env_nested_delimiter='_',
        env_nested_max_split=1,  # 只拆一层
    )
    llm: LLMConfig
```

这样 `GENERATION_LLM_API_KEY` 会被解析为 `llm.api_key`，而不是 `llm.api.key`。

### 方式二：多个独立的 Settings 类

对于完全独立的配置域，可以创建多个 `BaseSettings` 子类，各自管理自己的配置源。

```python
class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix='DB_')
    host: str = "localhost"
    port: int = 5432
    user: str
    password: str

class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix='APP_')
    name: str = "my_app"
    debug: bool = False
    port: int = 8000

# 各自独立加载
db_settings = DatabaseSettings()
app_settings = AppSettings()
```

**适用场景**：

- 不同模块的配置来源完全不同（比如数据库配置从 secrets 读，应用配置从环境变量读）
- 需要在不同时间点加载不同配置
- 微服务架构中，各服务独立管理自己的配置

### 方式三：组合模式（Settings 聚合）

在多个独立 Settings 类的基础上，再用一个顶层 Settings 类把它们聚合起来，方便全局访问。

```python
from pydantic import BaseModel

class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix='DB_')
    host: str = "localhost"
    port: int = 5432

class RedisSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix='REDIS_')
    host: str = "localhost"
    port: int = 6379

class Settings(BaseSettings):
    database: DatabaseSettings = DatabaseSettings()
    redis: RedisSettings = RedisSettings()
    app_name: str = "my_app"

settings = Settings()
```

注意：这种方式下，子 Settings 类各自用自己的 `env_prefix` 读取环境变量，顶层 Settings 不影响它们。

### 方式四：从配置文件拆分

pydantic-settings 支持从 TOML、YAML、JSON、pyproject.toml 等文件加载配置，可以按文件维度拆分。

**TOML 文件示例**：

```python
from pydantic_settings import (
    BaseSettings,
    PydanticBaseSettingsSource,
    SettingsConfigDict,
    TomlConfigSettingsSource,
)

class Nested(BaseModel):
    nested_field: str

class Settings(BaseSettings):
    foobar: str
    nested: Nested
    model_config = SettingsConfigDict(toml_file='config.toml')

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        return (TomlConfigSettingsSource(settings_cls),)
```

支持多文件合并：

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        toml_file=['config.default.toml', 'config.prod.toml']
    )
    # ...
```

后面的文件优先级更高，同名键覆盖前面的。可以用 `deep_merge=True` 做深度合并（需要在 source 实例化时设置，不能通过 `SettingsConfigDict`）。

### 方式五：自定义 Settings Source

如果内置的源不能满足需求，可以实现自定义的配置源。

```python
import json
from pathlib import Path
from typing import Any
from pydantic.fields import FieldInfo
from pydantic_settings import BaseSettings, PydanticBaseSettingsSource

class JsonConfigSettingsSource(PydanticBaseSettingsSource):
    def get_field_value(self, field: FieldInfo, field_name: str) -> tuple[Any, str, bool]:
        file_content = json.loads(Path('config.json').read_text())
        field_value = file_content.get(field_name)
        return field_value, field_name, False

    def prepare_field_value(self, field_name: str, field: FieldInfo, value: Any, value_is_complex: bool) -> Any:
        return value

    def __call__(self) -> dict[str, Any]:
        d: dict[str, Any] = {}
        for field_name, field in self.settings_cls.model_fields.items():
            field_value, field_key, value_is_complex = self.get_field_value(field, field_name)
            field_value = self.prepare_field_value(field_name, field, field_value, value_is_complex)
            if field_value is not None:
                d[field_key] = field_value
        return d

class Settings(BaseSettings):
    foobar: str

    @classmethod
    def settings_customise_sources(cls, settings_cls, init_settings, env_settings, dotenv_settings, file_secret_settings):
        return (init_settings, JsonConfigSettingsSource(settings_cls), env_settings)
```

### 方式六：按环境拆分（多环境配置）

通过不同的 `.env` 文件或配置文件实现环境隔离。

```python
import os

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=('.env', f'.env.{os.getenv("ENV", "dev")}'),
    )
    app_name: str
    database_url: str

settings = Settings()
```

加载顺序：`.env` → `.env.prod`（如果 `ENV=prod`），后者覆盖前者。

## 五、高级特性

### NoDecode：禁用 JSON 解析

默认情况下，复杂类型（list、dict、子模型）会从环境变量中按 JSON 解析。如果想用逗号分隔等自定义格式，可以用 `NoDecode`：

```python
from typing import Annotated
from pydantic import BeforeValidator
from pydantic_settings import BaseSettings, NoDecode

def split_comma(value):
    if isinstance(value, str):
        return [item.strip() for item in value.split(',')]
    return value

CommaSeparated = Annotated[list[str], NoDecode, BeforeValidator(split_comma)]

class Settings(BaseSettings):
    cors_origins: CommaSeparated = []

# export cors_origins='https://a.com, https://b.com'
```

### Secrets 管理

支持从本地密钥文件、Docker Secrets、AWS Secrets Manager、Azure Key Vault、Google Cloud Secret Manager 加载敏感配置。

```python
# Docker Secrets
class Settings(BaseSettings):
    model_config = SettingsConfigDict(secrets_dir='/run/secrets')
    db_password: str
```

### CLI 支持

可以直接把 Settings 模型变成 CLI 应用：

```python
from pydantic_settings import BaseSettings, CliApp, CliSubCommand, CliPositionalArg
from pydantic import BaseModel

class Init(BaseModel):
    directory: CliPositionalArg[str]
    def cli_cmd(self):
        print(f"init {self.directory}")

class Clone(BaseModel):
    repository: CliPositionalArg[str]
    def cli_cmd(self):
        print(f"clone {self.repository}")

class Git(BaseSettings, cli_parse_args=True):
    init: CliSubCommand[Init]
    clone: CliSubCommand[Clone]
    def cli_cmd(self):
        CliApp.run_subcommand(self)

CliApp.run(Git)
```

### 调试配置来源

设置 `PYDANTIC_SETTINGS_DEBUG=1` 可以查看每个来源加载到的具体值，方便排查配置问题：

```bash
PYDANTIC_SETTINGS_DEBUG=1 python your_app.py
```

## 六、各拆分方式优缺点对比

### 1. 嵌套模型（Nested Model）

| 优点 | 缺点 |
|------|------|
| 结构清晰，按模块组织 | 子模型是 `BaseModel` 不是 `BaseSettings`，无法独立加载 |
| 环境变量命名有层级感（`DB__HOST`） | 嵌套过深时环境变量名冗长 |
| 天然支持 JSON 格式传入整个子模型 | 子模型不能有自己独立的配置源 |
| 一个 Settings 实例即可访问所有配置 | 字段名冲突时需要用分隔符区分 |

**适用场景**：中等规模应用，配置项在几十个到上百个，按功能模块划分清晰。

### 2. 多个独立 Settings 类

| 优点 | 缺点 |
|------|------|
| 完全解耦，每个模块独立管理配置 | 配置分散在多个实例中，全局访问不便 |
| 各自可以有独立的 env_prefix 和来源 | 需要维护多个实例的生命周期 |
| 可以按需加载，减少不必要的 I/O | 跨模块配置引用不便 |
| 便于单元测试（只加载需要的配置） | 共享配置（如环境标识）需要重复定义 |

**适用场景**：微服务、大型应用中完全独立的子系统，或者配置来源差异很大的模块。

### 3. 组合模式（Settings 聚合）

| 优点 | 缺点 |
|------|------|
| 兼具独立加载和统一访问的好处 | 顶层实例化时会触发所有子 Settings 的加载 |
| 全局通过一个 `settings` 对象访问 | 子 Settings 各有各的 env_prefix，命名规则需要统一约定 |
| 子配置仍可独立使用和测试 | 层级关系不如嵌套模型直观 |

**适用场景**：大型应用，既需要模块化又需要统一入口。

### 4. 配置文件拆分（TOML/YAML/JSON）

| 优点 | 缺点 |
|------|------|
| 非技术人员也能看懂和修改 | 文件管理增加了部署复杂度 |
| 支持注释（TOML/YAML） | 需要确保配置文件随应用一起部署 |
| 复杂嵌套结构比环境变量更易读 | 敏感信息不适合放在明文配置文件中 |
| 多环境配置天然清晰（config.dev.toml 等） | 不如环境变量适合容器化部署 |

**适用场景**：配置项多且结构复杂，部署环境相对固定的场景。

### 5. 自定义 Settings Source

| 优点 | 缺点 |
|------|------|
| 灵活性极高，可以对接任意配置中心 | 需要编写和维护额外代码 |
| 可以接入公司内部配置系统 | 增加了调试和测试的复杂度 |
| 优先级机制可完全自定义 | 团队成员需要理解自定义源的行为 |

**适用场景**：需要对接 Nacos、Apollo、etcd 等配置中心的企业级应用。

### 6. 按环境拆分

| 优点 | 缺点 |
|------|------|
| 不同环境配置隔离清晰 | 需要管理多份配置文件 |
| 默认值 + 环境覆盖的模式很常用 | 敏感信息可能出现在 `.env` 文件中，需注意 gitignore |
| 配合 dotenv 使用简单直观 | 环境变量过多时管理成本上升 |

**适用场景**：几乎所有应用都需要（dev/test/prod 多环境）。

## 七、推荐实践

### 中小型项目

使用 **嵌套模型 + dotenv** 即可：

```python
# config.py
class Database(BaseModel):
    host: str
    port: int = 5432

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file='.env',
        env_nested_delimiter='__',
    )
    app_name: str = "my_app"
    database: Database
```

### 大型项目

使用 **组合模式 + 嵌套模型**：

```python
# config/database.py
class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix='DB_')
    host: str
    port: int = 5432

# config/redis.py
class RedisSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix='REDIS_')
    host: str
    port: int = 6379

# config/__init__.py
class Settings(BaseSettings):
    database: DatabaseSettings = DatabaseSettings()
    redis: RedisSettings = RedisSettings()
    app: AppSettings = AppSettings()
```

### 配置设计原则

1. **按职责拆分**：相关的配置放在一起，不相关的分开
2. **合理的嵌套层级**：一般 2-3 层为宜，过深会降低可读性
3. **敏感信息单独管理**：密码、密钥走 secrets 或环境变量，不要提交到代码库
4. **默认值要安全**：默认值应该对应开发环境，生产环境通过环境变量覆盖
5. **使用类型约束**：利用 Pydantic 的类型系统做校验，减少运行时错误
6. **配置集中导入**：整个应用通过统一的入口访问配置，避免散落各处

## 八、总结

Pydantic Settings 提供了强大而灵活的配置管理能力，拆分配置的方式多种多样，没有绝对的对错，关键在于根据项目规模和团队习惯选择合适的方式。

核心要点回顾：

- **嵌套模型**适合中小型项目，结构清晰、使用简单
- **多 Settings 类**适合大型项目，解耦彻底、灵活度高
- **配置文件**适合复杂结构，可读性好但部署稍复杂
- **自定义 Source**适合有特殊需求的企业级场景
- 无论哪种方式，都要注意**类型安全**、**敏感信息保护**和**可测试性**

根据实际情况组合使用这些方式，才能构建出既灵活又可维护的配置管理体系。
