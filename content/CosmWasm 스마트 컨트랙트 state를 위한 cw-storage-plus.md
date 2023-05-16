---
title: CosmWasm 스마트 컨트랙트 state를 위한 cw-storage-plus
---

# CosmWasm 스마트 컨트랙트 state를 위한 cw-storage-plus

## 요약

- cw-storage-plus는 스토리지 추상화 모듈로 `Item`, `Map`, `IndexedMap`, `Deque`가 있다.

## 내용

### Item

```rust
#[derive(Serialize, Deserialize, PartialEq, Debug)]
struct Config {
    pub owner: String,
    pub max_tokens: i32,
}

const CONFIG: Item<Config> = Item::new("config");

fn demo() -> StdResult<()> {
    let mut store = MockStorage::new();

    // may_load는 Option<T> 또는 None을 반환한다.
    // load는 T 또는 Err(StdError::NotFound{})을 반환한다.
    let empty = CONFIG.may_load(&store)?;
    assert_eq!(None, empty);
    let cfg = Config {
        owner: "admin".to_string(),
        max_tokens: 1234,
    };
    CONFIG.save(&mut store, &cfg)?;
    let loaded = CONFIG.load(&store)?;
    assert_eq!(cfg, loaded);

    // 클로저로 항목 업데이트 (읽기 및 쓰기 포기)
    let output = CONFIG.update(&mut store, |mut c| -> StdResult<_> {
        c.max_tokens *= 2;
        Ok(c)
    })?;
    assert_eq!(2468, output.max_tokens);

    // 오류
    let failed = CONFIG.update(&mut store, |_| -> StdResult<_> {
        Err(StdError::generic_err("failure mode"))
    });
    assert!(failed.is_err());

    let loaded = CONFIG.load(&store)?;
    let expected = Config {
        owner: "admin".to_string(),
        max_tokens: 2468,
    };
    assert_eq!(expected, loaded);

    // 삭제
    CONFIG.remove(&mut store);
    let empty = CONFIG.may_load(&store)?;
    assert_eq!(None, empty);

    Ok(())
}
```

- 데이터베이스 키를 제공함으로 데이터와 상호작용이 가능하다.
- `Item`은 `storage`안에 저장하지 않아 하나의 타입으로 충분하다.

### Map

```rust
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
struct Data {
    pub name: String,
    pub age: i32,
}

const PEOPLE: Map<&str, Data> = Map::new("people");

fn demo() -> StdResult<()> {
    let mut store = MockStorage::new();
    let data = Data {
        name: "John".to_string(),
        age: 32,
    };

    // 키로 읽고 저장
    let empty = PEOPLE.may_load(&store, "john")?;
    assert_eq!(None, empty);
    PEOPLE.save(&mut store, "john", &data)?;
    let loaded = PEOPLE.load(&store, "john")?;
    assert_eq!(data, loaded);

    // 없는 다른 키
    let missing = PEOPLE.may_load(&store, "jack")?;
    assert_eq!(None, missing);

    // 신규 또는 기존 키에 대한 업데이트
    let birthday = |d: Option<Data>| -> StdResult<Data> {
        match d {
            Some(one) => Ok(Data {
                name: one.name,
                age: one.age + 1,
            }),
            None => Ok(Data {
                name: "Newborn".to_string(),
                age: 0,
            }),
        }
    };

    let old_john = PEOPLE.update(&mut store, "john", birthday)?;
    assert_eq!(33, old_john.age);
    assert_eq!("John", old_john.name.as_str());

    let new_jack = PEOPLE.update(&mut store, "jack", birthday)?;
    assert_eq!(0, new_jack.age);
    assert_eq!("Newborn", new_jack.name.as_str());

    // 업데이트하면 스토어도 변경된다.
    assert_eq!(old_john, PEOPLE.load(&store, "john")?);
    assert_eq!(new_jack, PEOPLE.load(&store, "jack")?);

    // 제거하면 빈공간이 남음
    PEOPLE.remove(&mut store, "john");
    let empty = PEOPLE.may_load(&store, "john")?;
    assert_eq!(None, empty);

    Ok(())
}
```

- `BTreeMap`이며 입력된 값으로 `key-value` 조회가 가능하다.
- `iterator`를 이용하여 반복이 가능하다.
- simple(`&[u8]`) 또는 composite(`(&[u8], &[u8])`)로 PK 인덱싱이 된다. 고로 `Composite key`를 이용한 탐색도 가능하다.

#### Path

```rust
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
struct Data {
    pub name: String,
    pub age: i32,
}

const PEOPLE: Map<&str, Data> = Map::new("people");
const ALLOWANCE: Map<(&str, &str), u64> = Map::new("allow");

fn demo() -> StdResult<()> {
    let mut store = MockStorage::new();
    let data = Data {
        name: "John".to_string(),
        age: 32,
    };

    // 사용할 경로 생성
    let john = PEOPLE.key("john");

    let empty = john.may_load(&store)?;
    assert_eq!(None, empty);
    john.save(&mut store, &data)?;
    let loaded = john.load(&store)?;
    assert_eq!(data, loaded);
    john.remove(&mut store);
    let empty = john.may_load(&store)?;
    assert_eq!(None, empty);

    // 복합 키의 경우 key()에서 두 부분을 모두 사용
    let allow = ALLOWANCE.key(("owner", "spender"));
    allow.save(&mut store, &1234)?;
    let loaded = allow.load(&store)?;
    assert_eq!(1234, loaded);
    allow.update(&mut store, |x| Ok(x.unwrap_or_default() * 2))?;
    let loaded = allow.load(&store)?;
    assert_eq!(2468, loaded);

    Ok(())
}
```

- key에 접근 할 때 Map으로 부터 `Path`를 생성할 수 있다.
- `PEOPLE.load(&store, "jack") == PEOPLE.key("jack").load()`. `Map.key()`는 `Path`를 반환하여 `Item`가 동일한 인터페이스를 가진 키에 대한 계산된 경로를 다시 사용한다.
- `Composite key`를 두 번이라도 사용하는 모든 곳에 적용하는 것을 권장한다.

#### Prefix

```rust
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
struct Data {
    pub name: String,
    pub age: i32,
}

const PEOPLE: Map<&str, Data> = Map::new("people");
const ALLOWANCE: Map<(&str, &str), u64> = Map::new("allow");

fn demo() -> StdResult<()> {
    let mut store = MockStorage::new();

    let data = Data { name: "John".to_string(), age: 32 };
    PEOPLE.save(&mut store, "john", &data)?;
    let data2 = Data { name: "Jim".to_string(), age: 44 };
    PEOPLE.save(&mut store, "jim", &data2)?;

    // 모두 반복
    let all: StdResult<Vec<_>> = PEOPLE
        .range(&store, None, None, Order::Ascending)
        .collect();
    assert_eq!(
        all?,
        vec![("jim".to_vec(), data2), ("john".to_vec(), data.clone())]
    );

    // jim 다음에 무엇이 있는지 보여준다.
    let all: StdResult<Vec<_>> = PEOPLE
        .range(
            &store,
            Some(Bound::exclusive("jim")),
            None,
            Order::Ascending,
        )
        .collect();
    assert_eq!(all?, vec![("john".to_vec(), data)]);

    // 소유자가 다른 3개의 키에 저장 및 불러오기
    ALLOWANCE.save(&mut store, ("owner", "spender"), &1000)?;
    ALLOWANCE.save(&mut store, ("owner", "spender2"), &3000)?;
    ALLOWANCE.save(&mut store, ("owner2", "spender"), &5000)?;

    // 하나의 키로 모두 해결
    let all: StdResult<Vec<_>> = ALLOWANCE
        .prefix("owner")
        .range(&store, None, None, Order::Ascending)
        .collect();
    assert_eq!(
        all?,
        vec![("spender".to_vec(), 1000), ("spender2".to_vec(), 3000)]
    );

    // 두 항목의 범위로 가능
    let all: StdResult<Vec<_>> = ALLOWANCE
        .prefix("owner")
        .range(
            &store,
            Some(Bound::exclusive("spender")),
            Some(Bound::inclusive("spender2")),
            Order::Descending,
        )
        .collect();
    assert_eq!(all?, vec![("spender2".to_vec(), 3000)]);

    Ok(())
}
```

- map에서 특정 아이템을 가져오는 것 이외에 map을 반복하는 것도 가능하다.
- `map.prefix(k)`를 호출하여 prefix를 가져와서 사용한다.
- `bound`는 반복하려는 키에 type-safe 바운드를 구축하는 helper이며 raw`(Vec<u8>)` bound를 지원한다.

### IndexedMap

```rust
// 인덱스 정의
pub struct TokenIndexes<'a> {
  // 멀티 인덱스는 반복되는 값을 키로 가질 수 있다.
  // 가장 마지막 값이 PK로 쓰인다. (String)
  pub owner: MultiIndex<'a, Addr, TokenInfo, String>,
}

// 보일러 플레이트로 그대로 사용
// get_indexes로 인덱스가 쿼리될 수 있게 함
impl<'a> IndexList<TokenInfo> for TokenIndexes<'a> {
  fn get_indexes(&'_ self) -> Box<dyn Iterator<Item = &'_ dyn Index<TokenInfo>> + '_> {
    let v: Vec<&dyn Index<TokenInfo>> = vec![&self.owner];
    Box::new(v.into_iter())
  }
}

// TokenIndexs가 IndexList로 취급되며, IndexedMap 생성 중 파라미터 전달하는 것이 가능
pub fn tokens<'a>() -> IndexedMap<'a, &'a str, TokenInfo, TokenIndexes<'a>> {
  let indexes = TokenIndexes {
    owner: MultiIndex::new(
      |d: &TokenInfo| d.owner.clone(),
      "tokens",
      "tokens__owner",
    ),
  };
  IndexedMap::new("tokens", indexes)
}
```

- 사용 예제 (owner의 addr(string)으로 범위 안의 값을 찾는 코드)

```rust
    let limit = limit.unwrap_or(DEFAULT_LIMIT).min(MAX_LIMIT) as usize;
    let start = start_after.map(Bound::exclusive);
    let owner_addr = deps.api.addr_validate(&owner)?;

    let res: Result<Vec<_>, _> = tokens()
        .idx
        .owner
        .prefix(owner_addr)
        .range(deps.storage, start, None, Order::Ascending)
        .take(limit)
        .collect();
    let tokens = res?;
```

### Deque

```rust
#[derive(Serialize, Deserialize, PartialEq, Debug, Clone)]
struct Data {
    pub name: String,
    pub age: i32,
}

const DATA: Deque<Data> = Deque::new("data");

fn demo() -> StdResult<()> {
    let mut store = MockStorage::new();

    // 읽기 메서드는 래핑된 Option<T>를 반환하므로, 딕셔너리가 비어 있으면 None을 반환한다.
    let empty = DATA.front(&store)?;
    assert_eq!(None, empty);

    // some example entries
    let p1 = Data {
        name: "admin".to_string(),
        age: 1234,
    };
    let p2 = Data {
        name: "user".to_string(),
        age: 123,
    };

    // 큐 처럼 사용 가능
    DATA.push_back(&mut store, &p1)?;
    DATA.push_back(&mut store, &p2)?;

    let admin = DATA.pop_front(&mut store)?;
    assert_eq!(admin.as_ref(), Some(&p1));
    let user = DATA.pop_front(&mut store)?;
    assert_eq!(user.as_ref(), Some(&p2));

    // 스택 처럼 사용 가능
    DATA.push_back(&mut store, &p1)?;
    DATA.push_back(&mut store, &p2)?;

    let user = DATA.pop_back(&mut store)?;
    assert_eq!(user.as_ref(), Some(&p2));
    let admin = DATA.pop_back(&mut store)?;
    assert_eq!(admin.as_ref(), Some(&p1));

    // 반복 가능
    DATA.push_front(&mut store, &p1)?;
    DATA.push_front(&mut store, &p2)?;

    let all: StdResult<Vec<_>> = DATA.iter(&store)?.collect();
    assert_eq!(all?, [p2, p1]);

    // 인덱스에 직접 접근
    assert_eq!(DATA.get(&store, 0)?, Some(p2));
    assert_eq!(DATA.get(&store, 1)?, Some(p1));
    assert_eq!(DATA.get(&store, 3)?, None);

    Ok(())
}
```

- Rust std의 Deque의 스토리지 지원버전 처럼 작동하면 큐 또는 스택으로 사용할 수 있다.

## 참고

- [cw-storage-plus github](https://github.com/CosmWasm/cw-storage-plus)
- [cw-storage-plus cosmwasm docs](https://docs.cosmwasm.com/docs/smart-contracts/state/cw-plus)
- [docs.rs cw-storage-plus](https://docs.rs/cw-storage-plus/latest/cw_storage_plus/)
- [[CosmWasm을 이용한 코스모스 기반 스마트 컨트랙트 개발]]
