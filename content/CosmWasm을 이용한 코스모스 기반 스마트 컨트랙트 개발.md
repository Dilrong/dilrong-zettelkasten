---
title: CosmWasm을 이용한 코스모스 기반 스마트 컨트랙트 개발
---

# CosmWasm을 이용한 코스모스 기반 스마트 컨트랙트 개발

## 요약

- CosmWasm기반 스마트 컨트랙트는 `instantiate`, `execute`, `query` 진입점을 가지며 `state`에 값을 저장한다.
- 스마트 컨트랙트가 유효하기 위한 진입

## 내용

### 개요

- CosmWasm은 멀티체인 스마트 컨트랙트 플랫폼으로 Cosmos 생태계에서 Web Assembly(Wasm)을 이용한 스마트컨트랙트 플랫폼이다.
- 코스모스 생태계의 대표적인 예는 코스모스, 오스모시스, 라인 블록체인, 컴투스 엑스플라 등이 텐더민트 기반의 메인넷을 가진 블록체인이 있다.

### 환경 설정

- CosmWasm는 Rust 기반으로 [Rust 설치](https://www.rust-lang.org/tools/install)가 필요하다.

```zsh
cargo install cargo-generate --features vendored-openssl
cargo install cargo-run-script
cargo generate --git https://github.com/CosmWasm/cw-template.git --name PROJECT_NAME
```

- 위 명령어로 CosmWasm 스마트 컨트랙트 템플릿(`cw-template`)을 받을 수 있으며 `cw-template` 기본으로 스마트 컨트랙트를 만든다.

### 스마트 컨트랙트 개요

- 메시지가 컨트랙트로 전송되면 `entry point`함수가 호출된다.
- 스마트 컨트랙트의 대부분은 기본적으로 3개의 진입점을 만들어서 사용한다.
  - `instantiate`: 스마트 컨트랙트 수명동안 한 번 호출되며, 컨트랙트의 생성자이다.
  - `execute`: 컨트랙트 상태를 수정할 수 있는 메시지를 처리하기 위한 것으로, 실제 작업을 수행하는 데 사용된다.
  - `query`: 컨트랙트에서 정보를 요청하는 메시지를 처리하는데 사용된다. 데이터베이스 쿼리처럼 사용된다.

### Escrow Example

- deus-labs의 [escrow](https://github.com/deus-labs/cw-contracts/tree/main/contracts/escrow)스마트 컨트랙트를 예제로 설명한다.
- 예제는 간단한 일회용 안전거래 컨트랙트로 네이티브 토큰을 보유할 수 있는 컨트랙트를 생성하고 중재자에게 미리 정의된 수혜자에게 토큰을 가져올 수 있는 권한을 부여한다. 추가로 만료 시간에 도달하면 더 이상 가져올 수 없고 원래 펀딩자에게만 반환할 수 있게 된다.
- 프로젝트 구조는 다음과 같다.
  - src
    - contract.rs: 컨트랙트 로직 정의
    - error.rs: 컨트랙트 오류 정의
    - lib.rs: 모듈 import
    - msg.rs: 컨트랙트 진입 메시지 정의
    - state.rs: 컨트랙트 상태 정의
  - schema: 스마트 컨트랙트와 인터랙트하는 메시지 형태 JSON
  - tests: 통합 테스트 코드

#### Msg, 메시지

```rust
/// msg.rs
use cosmwasm_schema::{cw_serde, QueryResponses};
use cosmwasm_std::{Addr, Coin};
use cw_utils::Expiration;

#[cw_serde]
pub struct InstantiateMsg {
    pub arbiter: String,
    pub recipient: String,
    pub expiration: Option<Expiration>,
}

#[cw_serde]
pub enum ExecuteMsg {
    Approve {
        quantity: Option<Vec<Coin>>,
    },
    Refund {},
}

#[cw_serde]
#[derive(QueryResponses)]
pub enum QueryMsg {
    #[returns(ArbiterResponse)]
    Arbiter {},
}

#[cw_serde]
pub struct ArbiterResponse {
    pub arbiter: Addr,
}
```

- 스마트 컨트랙트에 트랜잭션을 보낼 때 담길 메시지를 정의한 모듈으로 진입 메세지를 정의한다.

#### Instantiate, 생성자

```rust
/// contract.rs

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    let config = Config {
        arbiter: deps.api.addr_validate(&msg.arbiter)?,
        recipient: deps.api.addr_validate(&msg.recipient)?,
        source: info.sender,
        expiration: msg.expiration,
    };

    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;

    if let Some(expiration) = msg.expiration {
        if expiration.is_expired(&env.block) {
            return Err(ContractError::Expired { expiration });
        }
    }
    CONFIG.save(deps.storage, &config)?;
    Ok(Response::default())
}
```

- `instantiate`는 스마트 컨트랙트가 유효하기 위해 필요한 진입점으로 구조는 다음과 같다.
  - `deps`: `DepsMut`는 통신하기 위한 유틸리티 타입으로 컨트랙트 상태를 쿼리 및 업데이트한다.
  - `env`: `Env`는 메시지를 실행할 때 블록체인 상태를 나타내는 객체로 체인 높이와 아이디, 현재 타임 스탬프, 호출된 컨트랙트 주소를 포함한다.
  - `info`: `MessageInfo`는 메시지를 전송한 주소, 메시지에 함께 전송된 토큰 등 메타정보가 포함되어 있다.
  - `msg`: `Empty` 실행을 트리거하는 메시지이다.
- `#[cfg_attr(not(feature = "library"), entry_point)]` 는 어트리뷰트 (attribute)이며 컴파일이 되는 조건을 명시한다.
- library feature가 없다면 entry_point가 적용되어 컴파일이 된다. 라는 어트리뷰트이다.

#### Execute, 실행

```rust
/// contract.rs

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::Approve { quantity } => execute_approve(deps, env, info, quantity),
        ExecuteMsg::Refund {} => execute_refund(deps, env, info),
    }
}

fn execute_approve(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    quantity: Option<Vec<Coin>>,
) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    if info.sender != config.arbiter {
        return Err(ContractError::Unauthorized {});
    }

    // throws error if the contract is expired
    if let Some(expiration) = config.expiration {
        if expiration.is_expired(&env.block) {
            return Err(ContractError::Expired { expiration });
        }
    }

    let amount = if let Some(quantity) = quantity {
        quantity
    } else {
        deps.querier.query_all_balances(&env.contract.address)?
    };
    Ok(send_tokens(config.recipient, amount, "approve"))
}

fn execute_refund(deps: DepsMut, env: Env, _info: MessageInfo) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;

    if let Some(expiration) = config.expiration {
        if !expiration.is_expired(&env.block) {
            return Err(ContractError::NotExpired {});
        }
    } else {
        return Err(ContractError::NotExpired {});
    }

    let balance = deps.querier.query_all_balances(&env.contract.address)?;
    Ok(send_tokens(config.source, balance, "refund"))
}


fn send_tokens(to_address: Addr, amount: Vec<Coin>, action: &str) -> Response {
    Response::new()
        .add_message(BankMsg::Send {
            to_address: to_address.clone().into(),
            amount,
        })
        .add_attribute("action", action)
        .add_attribute("to", to_address)
}
```

- state값을 변경하는 로직부분으로 승인(Approve) 그리고 환불(Refund)의 msg를 받아서 해당 함수에 진입한다.
- Approve는 지출한도 승인을 받는 함수이다.
- Refund는 expiration된 코인을 돌려받는 함수이다.

#### Query, 쿼리

```rust
/// contract.rs

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    match msg {
        QueryMsg::Arbiter {} => to_binary(&query_arbiter(deps)?),
    }
}

fn query_arbiter(deps: Deps) -> StdResult<ArbiterResponse> {
    let config = CONFIG.load(deps.storage)?;
    let addr = config.arbiter;
    Ok(ArbiterResponse { arbiter: addr })
}
```

- 컨트랙트의 storage를 조회하여 state를 사용자에게 반환한다.
- `query_arbiter`는 config에서 arbiter을 정보를 반환하는 함수이다.

#### State, 상태

```rust
/// state.rs
use cosmwasm_schema::cw_serde;
use cosmwasm_std::Addr;
use cw_storage_plus::Item;
use cw_utils::Expiration;

#[cw_serde]
pub struct Config {
    pub arbiter: Addr,
    pub recipient: Addr,
    pub source: Addr,
    pub expiration: Option<Expiration>,
}

pub const CONFIG_KEY: &str = "config";
pub const CONFIG: Item<Config> = Item::new(CONFIG_KEY);
```

- 컨트랙트의 상태로 storage의 해당 값을 저장한다.
- CosmWasm 스마트 컨트랙트는 바이트 기반의 키-밸류 스토어인 `LevelDB`에 영구적인 값, State 저장이 가능하다.
- `cw-storage-plus`는 CosmWasm을 위한 스토리지 추상화 모듈이다.
- `cw_serde`는 직렬화에서 들어가는 규칙을 정의해둔 매크로로 `#[derive(Serialize, Deserialize, PartialEq, Debug, Clone, JsonSchema)] #[serde(rename_all = "snake_case")]` 과 같은 효과를 낸다.

#### Test, 테스트

```rust
/// contract.rs

#[cfg(test)]
mod tests {
    use super::*;
    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};
    use cosmwasm_std::{coins, CosmosMsg, Timestamp};
    use cw_utils::Expiration;

    fn init_msg_expire_by_height(expiration: Option<Expiration>) -> InstantiateMsg {
        InstantiateMsg {
            arbiter: String::from("verifies"),
            recipient: String::from("benefits"),
            expiration,
        }
    }
    ...
    #[test]
    fn init_and_query() {
        let mut deps = mock_dependencies();

        let arbiter = Addr::unchecked("arbiters");
        let recipient = Addr::unchecked("receives");
        let creator = Addr::unchecked("creates");
        let msg = InstantiateMsg {
            arbiter: arbiter.clone().into(),
            recipient: recipient.into(),
            expiration: None,
        };
        let mut env = mock_env();
        env.block.height = 876;
        env.block.time = Timestamp::from_seconds(0);
        let info = mock_info(creator.as_str(), &[]);
        let res = instantiate(deps.as_mut(), env, info, msg).unwrap();
        assert_eq!(0, res.messages.len());

        // now let's query
        let query_response = query_arbiter(deps.as_ref()).unwrap();
        assert_eq!(query_response.arbiter, arbiter);
    }
}
```

- 컨트랙트가 올바른 동작을 하는지 테스트한다.
- `contract.rs`의 `init_and_query()`을 예시로 설명하면 다음과 같다.
  - `mock_dependencies()`와 `mock_env()`를 사용하여 목업 환경을 만든다.
  - `init_msg_expire_by_height` 을 메시지를 만들고 `instantiate`으로 컨트랙트 초기화를 한다.

## 참고

- [CosmWasm book](https://book.cosmwasm.com/index.html)
- https://medium.com/cosmwasm/its-showtime-%ED%95%9C%EA%B5%AD%EC%96%B4-%EB%B2%88%EC%97%AD-%EB%B2%84%EC%A0%84-a62f3b84cfb1
