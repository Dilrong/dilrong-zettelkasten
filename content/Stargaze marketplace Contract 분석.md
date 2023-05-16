---
title: Stargaze marketplace Contract 분석
---

# Stargaze marketplace Contract 분석

## 요약

- `sudo`, `execute` 로 기능이 분할되어 있으며 `ask`, `bid`, `collection bid`로 총 3개의 기능이 있다.
- 경매, 고정가 2개의 판매 타입이 있으며 `ask`, `bid`, `collection` 3개의 상태값으로 나눠진다.

## 내용

### 구조

- src
  - bin: 빌드시 생성되는 스키마 폴
  - testing: 테스트 모음
  - error.rs: 컨트랙트 오류 정의
  - execute.rs: 컨트랙트 로직 정의
  - helpers.rs: 컨트랙트 유틸 정의
  - lib.rs: 모듈 import
  - msg.rs: 컨트랙트 진입 메시지
  - query.rs: 컨트랙트 쿼리 정의
  - state.rs: 컨트랙트 상태 정의
  - sudo.rs: 컨트랙트 어드민 기능
  - testing.rs: 테스트 모듈 import

### 주요 기능

#### sudo

- 어드민(sudo)는 마켓 설정을 수정하고 마켓 오퍼레이터를 지정 할 수 있다.
- `sudo_update_params`: 마켓 설정 수정
- `sudo_(add, remove)_operator`: 마켓 오퍼레이터 추가, 삭제
- `sudo_(add, remove)_~_hook`: ~해당 훅 추가, 삭제

#### execute

- 사용자는 요청, 입찰, 컬렉션 입찰에 대한 CRUD를 하고 거래를 할 수 있다.
- 오퍼레이터는 마켓 관리를 위해서 요청 동기화, 오래된 요청, 입찰, 컬렉션 입찰을 삭제 할 수 있다.
- `execute_(set, remove, update)_ask`: 요청(ask) 추가, 삭제, 업데이트
- `execute_(set, remove, accept, reject)_bid`: 입찰(bid) 추가, 삭제, 수락, 거절
- `execute_(set, remove, accept)_collection_bid`: 컬렉션 입찰(bid) 설정, 삭제, 수락
  - 컬렉션 전체에 대한 제안로 하나의 nft에 대한 제안이 아님
- `execute_sync_ask`: 요청 동기화
- `execute_remove_stable_(ask, bid, collection_bid)`: 오래된 요청, 입찰, 컬렉션 입찰 삭제
- `finalize_sale`: 구매처리

### msg.rs

#### InstantiateMsg

```rust
#[cw_serde]
pub struct InstantiateMsg {
    /// 낙찰 수수료
    /// 0.25% = 25, 0.5% = 50, 1% = 100, 2.5% = 250
    pub trading_fee_bps: u64,
    /// Asks 유효시간 범위(초)
    pub ask_expiry: ExpiryRange,
    /// Bids 유효시간 범위(초)
    pub bid_expiry: ExpiryRange,
    /// 오퍼레이터
    pub operators: Vec<String>,
    /// 매출 계산을 위한 에어드랍 클레임 계약 주소
    pub sale_hook: Option<String>,
    /// 파운더에 대한 최대 수수료
    pub max_finders_fee_bps: u64,
    /// 최소가격
    pub min_price: Uint128,
    /// 입찰이 만료된 유효 기간(초)
    pub stale_bid_duration: Duration,
    /// 오래된 입찰 제거의 보상
    pub bid_removal_reward_bps: u64,
    /// 스팸을 막기 위한 리스팅 수수료
    pub listing_fee: Uint128,
}
```

#### ExecuteMsg

```rust
#[cw_serde]
pub enum ExecuteMsg {
    /// Ask 생성 (NFT 리스팅)
    SetAsk {
        sale_type: SaleType,
        collection: String,
        token_id: TokenId,
        price: Coin,
        funds_recipient: Option<String>,
        reserve_for: Option<String>,
        finders_fee_bps: Option<u64>,
        expires: Timestamp,
    },
    /// Ask 삭제
    RemoveAsk {
        collection: String,
        token_id: TokenId,
    },
    /// Ask 가격 업데이트
    UpdateAskPrice {
        collection: String,
        token_id: TokenId,
        price: Coin,
    },
    /// 기존 Ask에 입찰
    SetBid {
        collection: String,
        token_id: TokenId,
        expires: Timestamp,
        sale_type: SaleType,
        finder: Option<String>,
        finders_fee_bps: Option<u64>,
    },
    /// 바로 구매
    BuyNow {
        collection: String,
        token_id: TokenId,
        expires: Timestamp,
        finder: Option<String>,
        finders_fee_bps: Option<u64>,
    },
    /// 입찰 취소
    RemoveBid {
        collection: String,
        token_id: TokenId,
    },
    /// 입찰 수락
    AcceptBid {
        collection: String,
        token_id: TokenId,
        bidder: String,
        finder: Option<String>,
    },
    /// 입찰 거절
    RejectBid {
        collection: String,
        token_id: TokenId,
        bidder: String,
    },
    /// 전체 컬렉션에 대해 입찰 (제한 주문)
    SetCollectionBid {
        collection: String,
        expires: Timestamp,
        finders_fee_bps: Option<u64>,
    },
    /// 전체 컬렉션에서 입찰 제거
    RemoveCollectionBid { collection: String },
    /// Accept a collection bid
    AcceptCollectionBid {
        collection: String,
        token_id: TokenId,
        bidder: String,
        finder: Option<String>,
    },
    /// NFT가 전송될 때 요청의 활성 상태를 변경하는 권한이 있는 작업
    SyncAsk {
        collection: String,
        token_id: TokenId,
    },
    /// 오래되거나 잘못된 요청 제거하는 권한이 있는 작업
    RemoveStaleAsk {
        collection: String,
        token_id: TokenId,
    },
    /// 오래된 입찰을 제거하는 권한이 있는 작업
    RemoveStaleBid {
        collection: String,
        token_id: TokenId,
        bidder: String,
    },
    /// 오래된 수집 입찰을 제거하는 권한이 있는 작업
    RemoveStaleCollectionBid { collection: String, bidder: String },
}
```

#### SudoMsg

```rust
pub enum SudoMsg {
    /// 컨트랙트 Params 수정
    UpdateParams {
        trading_fee_bps: Option<u64>,
        ask_expiry: Option<ExpiryRange>,
        bid_expiry: Option<ExpiryRange>,
        operators: Option<Vec<String>>,
        max_finders_fee_bps: Option<u64>,
        min_price: Option<Uint128>,
        stale_bid_duration: Option<u64>,
        bid_removal_reward_bps: Option<u64>,
        listing_fee: Option<Uint128>,
    },
    /// 오퍼레이터 추가
    AddOperator { operator: String },
    /// 오퍼레이터 삭제
    RemoveOperator { operator: String },
    /// 모든 Asks에 대한 알림을 받을 훅 추가
    AddAskHook { hook: String },
    /// 모든 입찰에 대한 알림을 받을 훅 추가
    AddBidHook { hook: String },
    /// Ask 훅 삭제
    RemoveAskHook { hook: String },
    /// 입찰 훅 삭제
    RemoveBidHook { hook: String },
    /// 모든 거래에 대한 훅 추가
    AddSaleHook { hook: String },
    /// 거래에 대한 훅 삭제
    RemoveSaleHook { hook: String },
}
```

#### QueryMsg

```rust
pub enum QueryMsg {
    /// 컬렉션 리스트를 보여준다.
    #[returns(CollectionsResponse)]
    Collections {
        start_after: Option<Collection>,
        limit: Option<u32>,
    },
    /// 특정NFT에 대한 현재 ask를 보여준다.
    #[returns(AsksResponse)]
    Ask {
        collection: Collection,
        token_id: TokenId,
    },
    /// 컬렉션의 모든 ask를 보여준다.
    #[returns(AsksResponse)]
    Asks {
        collection: Collection,
        include_inactive: Option<bool>,
        start_after: Option<TokenId>,
        limit: Option<u32>,
    },
    /// 컬렉션의 모든 ask를 역순으로 보여준다.
    #[returns(AsksResponse)]
    ReverseAsks {
        collection: Collection,
        include_inactive: Option<bool>,
        start_before: Option<TokenId>,
        limit: Option<u32>,
    },
    /// 컬렉션의 모든 ask를 가격순으로 보여준다.
    #[returns(AsksResponse)]
    AsksSortedByPrice {
        collection: Collection,
        include_inactive: Option<bool>,
        start_after: Option<AskOffset>,
        limit: Option<u32>,
    },
    /// 컬렉션의 모든 ask를 가격 역순으로 보여준다.
    #[returns(AsksResponse)]
    ReverseAsksSortedByPrice {
        collection: Collection,
        include_inactive: Option<bool>,
        start_before: Option<AskOffset>,
        limit: Option<u32>,
    },
    /// 모든 ask 카운트를 보여준다.
    #[returns(AskCountResponse)]
    AskCount { collection: Collection },
    /// 판매자 별 모든 ask를 보여준다.
    #[returns(AsksResponse)]
    AsksBySeller {
        seller: Seller,
        include_inactive: Option<bool>,
        start_after: Option<CollectionOffset>,
        limit: Option<u32>,
    },
    /// 특정 입찰에 대한 정보를 보여준다.
    #[returns(BidResponse)]
    Bid {
        collection: Collection,
        token_id: TokenId,
        bidder: Bidder,
    },
    /// 입찰자별 모든 입찰을 보여준다.
    #[returns(BidsResponse)]
    BidsByBidder {
        bidder: Bidder,
        start_after: Option<CollectionOffset>,
        limit: Option<u32>,
    },
    /// 입찰자별 모든 입찰을 만기일 순으로 보여준다.
    #[returns(BidsResponse)]
    BidsByBidderSortedByExpiration {
        bidder: Bidder,
        start_after: Option<CollectionOffset>,
        limit: Option<u32>,
    },
    /// 특정 NFT의 모든 입찰을 보여준다.
    #[returns(BidsResponse)]
    Bids {
        collection: Collection,
        token_id: TokenId,
        start_after: Option<Bidder>,
        limit: Option<u32>,
    },
    /// 컬렉션의 모든 입찰을 가격 순으로 보여준다.
    #[returns(BidsResponse)]
    BidsSortedByPrice {
        collection: Collection,
        start_after: Option<BidOffset>,
        limit: Option<u32>,
    },
    /// 컬렉션의 모든 입찰을 가격 역순으로 보여준다.
    #[returns(BidsResponse)]
    ReverseBidsSortedByPrice {
        collection: Collection,
        start_before: Option<BidOffset>,
        limit: Option<u32>,
    },
    /// 특정 컬렉션 입찰 데이터를 보여준다.
    #[returns(CollectionBidResponse)]
    CollectionBid {
        collection: Collection,
        bidder: Bidder,
    },
    /// 모든 컬렉션 입찰자 기준 입찰을 보여준다.
    #[returns(CollectionBidResponse)]
    CollectionBidsByBidder {
        bidder: Bidder,
        start_after: Option<CollectionOffset>,
        limit: Option<u32>,
    },
    /// 모든 컬렉션 입찰자 기준 입찰을 마감일 순으로 보여준다.
    #[returns(CollectionBidResponse)]
    CollectionBidsByBidderSortedByExpiration {
        bidder: Collection,
        start_after: Option<CollectionBidOffset>,
        limit: Option<u32>,
    },
    /// 모든 컬렉션 입찰자 기준 입찰을 가격을 순으로 보여준다.
    #[returns(CollectionBidResponse)]
    CollectionBidsSortedByPrice {
        collection: Collection,
        start_after: Option<CollectionBidOffset>,
        limit: Option<u32>,
    },
    /// 모든 컬렉션 입찰자 기준 입찰을 가격을 역순으로 보여준다.
    #[returns(CollectionBidResponse)]
    ReverseCollectionBidsSortedByPrice {
        collection: Collection,
        start_before: Option<CollectionBidOffset>,
        limit: Option<u32>,
    },
    /// 등록된 훅을 보여준다.
    #[returns(HooksResponse)]
    AskHooks {},
    /// 등록된 훅을 보여준다.
    #[returns(HooksResponse)]
    BidHooks {},
    /// 등록된 훅을 보여준다.
    #[returns(HooksResponse)]
    SaleHooks {},
    /// 등록된 훅을 보여준다.
    #[returns(ParamsResponse)]
    Params {},
}
```

### state.rs

#### SudoParams, 어드민 설정값

```rust
pub struct SudoParams {
    /// 낙찰 수수료
    /// 0.25% = 25, 0.5% = 50, 1% = 100, 2.5% = 250
    pub trading_fee_bps: u64,
    /// Asks 유효시간 범위(초)
    pub ask_expiry: ExpiryRange,
    /// Bids 유효시간 범위(초)
    pub bid_expiry: ExpiryRange,
    /// 오퍼레이터
    pub operators: Vec<String>,
    /// 매출 계산을 위한 에어드랍 클레임 계약 주소
    pub sale_hook: Option<String>,
    /// 파운더에 대한 최대 수수료
    pub max_finders_fee_bps: u64,
    /// 최소가격
    pub min_price: Uint128,
    /// 입찰이 만료된 유효 기간(초)
    pub stale_bid_duration: Duration,
    /// 오래된 입찰 제거의 보상
    pub bid_removal_reward_bps: u64,
    /// 스팸을 막기 위한 리스팅 수수료
    pub listing_fee: Uint128,
}
```

#### SaleType, 판매 종류

```rust
#[cw_serde]
pub enum SaleType {
    FixedPrice,
    Auction,
}

impl fmt::Display for SaleType {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
	        // 고정가
            SaleType::FixedPrice => write!(f, "fixed_price"),
            // 경매
            SaleType::Auction => write!(f, "auction"),
        }
    }
}
```

#### Ask, 요청

```rust
#[cw_serde]
pub struct Ask {
	// 판매 종류
    pub sale_type: SaleType,
	// 컬렉션 주소
    pub collection: Addr,
    // 토큰 아이디
    pub token_id: TokenId,
    // 판매자 주소
    pub seller: Addr,
    // 가격
    pub price: Uint128,
    // 로열티 받는 주소
    pub funds_recipient: Option<Addr>,
    // 예비 주소
    pub reserve_for: Option<Addr>,
    // 파운더 피
    pub finders_fee_bps: Option<u64>,
    // 마감일
    pub expires_at: Timestamp,
    // 활성화 여부
    pub is_active: bool,
}

// PK는 (collection, token_id)
pub type AskKey = (Addr, TokenId);

/// 토큰 인덱스 정의
pub struct AskIndicies<'a> {
    pub collection: MultiIndex<'a, Addr, Ask, AskKey>,
    pub collection_price: MultiIndex<'a, (Addr, u128), Ask, AskKey>,
    pub seller: MultiIndex<'a, Addr, Ask, AskKey>,
}

// 인덱스리스트 보일러플레이트
impl<'a> IndexList<Ask> for AskIndicies<'a> {
    fn get_indexes(&'_ self) -> Box<dyn Iterator<Item = &'_ dyn Index<Ask>> + '_> {
        let v: Vec<&dyn Index<Ask>> = vec![&self.collection, &self.collection_price, &self.seller];
        Box::new(v.into_iter())
    }
}

// 멀티인덱스 쿼리 설정
pub fn asks<'a>() -> IndexedMap<'a, AskKey, Ask, AskIndicies<'a>> {
    let indexes = AskIndicies {
        collection: MultiIndex::new(
            |_pk: &[u8], d: &Ask| d.collection.clone(),
            "asks",
            "asks__collection",
        ),
        collection_price: MultiIndex::new(
            |_pk: &[u8], d: &Ask| (d.collection.clone(), d.price.u128()),
            "asks",
            "asks__collection_price",
        ),
        seller: MultiIndex::new(
            |_pk: &[u8], d: &Ask| d.seller.clone(),
            "asks",
            "asks__seller",
        ),
    };
    IndexedMap::new("asks", indexes)
}
```

#### Bid, 입찰

```rust
/// 입찰
#[cw_serde]
pub struct Bid {
	// 컬렉션 주소
    pub collection: Addr,
	// 토큰 id
    pub token_id: TokenId,
    // 비더 주소
    pub bidder: Addr,
    // 가격
    pub price: Uint128,
    // 파운더 수수료
    pub finders_fee_bps: Option<u64>,
    // 입찰 마감일
    pub expires_at: Timestamp,
}

/// 입찰 인덱스
pub struct BidIndicies<'a> {
    pub collection: MultiIndex<'a, Addr, Bid, BidKey>,
    pub collection_token_id: MultiIndex<'a, (Addr, TokenId), Bid, BidKey>,
    pub collection_price: MultiIndex<'a, (Addr, u128), Bid, BidKey>,
    pub bidder: MultiIndex<'a, Addr, Bid, BidKey>,
    pub bidder_expires_at: MultiIndex<'a, (Addr, u64), Bid, BidKey>,
}

impl<'a> IndexList<Bid> for BidIndicies<'a> {
    fn get_indexes(&'_ self) -> Box<dyn Iterator<Item = &'_ dyn Index<Bid>> + '_> {
        let v: Vec<&dyn Index<Bid>> = vec![
            &self.collection,
            &self.collection_token_id,
            &self.collection_price,
            &self.bidder,
            &self.bidder_expires_at,
        ];
        Box::new(v.into_iter())
    }
}

pub fn bids<'a>() -> IndexedMap<'a, BidKey, Bid, BidIndicies<'a>> {
    let indexes = BidIndicies {
        collection: MultiIndex::new(
            |_pk: &[u8], d: &Bid| d.collection.clone(),
            "bids",
            "bids__collection",
        ),
        collection_token_id: MultiIndex::new(
            |_pk: &[u8], d: &Bid| (d.collection.clone(), d.token_id),
            "bids",
            "bids__collection_token_id",
        ),
        collection_price: MultiIndex::new(
            |_pk: &[u8], d: &Bid| (d.collection.clone(), d.price.u128()),
            "bids",
            "bids__collection_price",
        ),
        bidder: MultiIndex::new(
            |_pk: &[u8], d: &Bid| d.bidder.clone(),
            "bids",
            "bids__bidder",
        ),
        bidder_expires_at: MultiIndex::new(
            |_pk: &[u8], d: &Bid| (d.bidder.clone(), d.expires_at.seconds()),
            "bids",
            "bids__bidder_expires_at",
        ),
    };
    IndexedMap::new("bids", indexes)
}
```

#### CollectionBid, 컬렉션 입찰 내역

```rust
#[cw_serde]
pub struct CollectionBid {
    pub collection: Addr,
    pub bidder: Addr,
    pub price: Uint128,
    pub finders_fee_bps: Option<u64>,
    pub expires_at: Timestamp,
}

impl Order for CollectionBid {
    fn expires_at(&self) -> Timestamp {
        self.expires_at
    }
}

/// 입찰의 pk키: (collection, token_id, bidder)
pub type CollectionBidKey = (Addr, Addr);
/// 입찰 키 생성
pub fn collection_bid_key(collection: &Addr, bidder: &Addr) -> CollectionBidKey {
    (collection.clone(), bidder.clone())
}

/// 컬렉션 입찰내역에 접근하기 위한 인덱스
pub struct CollectionBidIndicies<'a> {
    pub collection: MultiIndex<'a, Addr, CollectionBid, CollectionBidKey>,
    pub collection_price: MultiIndex<'a, (Addr, u128), CollectionBid, CollectionBidKey>,
    pub bidder: MultiIndex<'a, Addr, CollectionBid, CollectionBidKey>,
    pub bidder_expires_at: MultiIndex<'a, (Addr, u64), CollectionBid, CollectionBidKey>,
}

impl<'a> IndexList<CollectionBid> for CollectionBidIndicies<'a> {
    fn get_indexes(&'_ self) -> Box<dyn Iterator<Item = &'_ dyn Index<CollectionBid>> + '_> {
        let v: Vec<&dyn Index<CollectionBid>> = vec![
            &self.collection,
            &self.collection_price,
            &self.bidder,
            &self.bidder_expires_at,
        ];
        Box::new(v.into_iter())
    }
}

pub fn collection_bids<'a>(
) -> IndexedMap<'a, CollectionBidKey, CollectionBid, CollectionBidIndicies<'a>> {
    let indexes = CollectionBidIndicies {
        collection: MultiIndex::new(
            |_pk: &[u8], d: &CollectionBid| d.collection.clone(),
            "col_bids",
            "col_bids__collection",
        ),
        collection_price: MultiIndex::new(
            |_pk: &[u8], d: &CollectionBid| (d.collection.clone(), d.price.u128()),
            "col_bids",
            "col_bids__collection_price",
        ),
        bidder: MultiIndex::new(
            |_pk: &[u8], d: &CollectionBid| d.bidder.clone(),
            "col_bids",
            "col_bids__bidder",
        ),
        bidder_expires_at: MultiIndex::new(
            |_pk: &[u8], d: &CollectionBid| (d.bidder.clone(), d.expires_at.seconds()),
            "col_bids",
            "col_bids__bidder_expires_at",
        ),
    };
    IndexedMap::new("col_bids", indexes)
}
```

### query.rs

- msg와 동일하며 해당 state 반환

### execute.rs

#### execute_set_ask

```rust
pub fn execute_set_ask(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    ask_info: AskInfo,
) -> Result<Response, ContractError> {
    let AskInfo {
        sale_type,
        collection,
        token_id,
        price,
        funds_recipient,
        reserve_for,
        finders_fee_bps,
        expires,
    } = ask_info;

    price_validate(deps.storage, &price)?;
    // 가격 확인
    if price.amount.u128() > MAX_FIXED_PRICE_ASK_AMOUNT {
        return Err(ContractError::PriceTooHigh(price.amount));
    }

    only_owner(deps.as_ref(), &info, &collection, token_id)?;
    only_tradable(deps.as_ref(), &env.block, &collection)?;

    // 토큰 이동 권한이 있는지 확인
    Cw721Contract::<Empty, Empty>(collection.clone(), PhantomData, PhantomData).approval(
        &deps.querier,
        token_id.to_string(),
        env.contract.address.to_string(),
        None,
    )?;

    let params = SUDO_PARAMS.load(deps.storage)?;
    params.ask_expiry.is_valid(&env.block, expires)?;

    // 리스팅 수수료 확인
    let listing_fee = may_pay(&info, NATIVE_DENOM)?;
    if listing_fee != params.listing_fee {
        return Err(ContractError::InvalidListingFee(listing_fee));
    }

    if let Some(fee) = finders_fee_bps {
        if Decimal::percent(fee) > params.max_finders_fee_percent {
            return Err(ContractError::InvalidFindersFeeBps(fee));
        };
    }

    let mut event = Event::new("set-ask")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("sale_type", sale_type.to_string());

    if let Some(address) = reserve_for.clone() {
        if address == info.sender {
            return Err(ContractError::InvalidReserveAddress {
                reason: "cannot reserve to the same address".to_string(),
            });
        }
        if sale_type != SaleType::FixedPrice {
            return Err(ContractError::InvalidReserveAddress {
                reason: "can only reserve for fixed_price sales".to_string(),
            });
        }
        event = event.add_attribute("reserve_for", address.to_string());
    }

    let seller = info.sender;
    let ask = Ask {
        sale_type,
        collection,
        token_id,
        seller: seller.clone(),
        price: price.amount,
        funds_recipient,
        reserve_for,
        finders_fee_bps,
        expires_at: expires,
        is_active: true,
    };
    store_ask(deps.storage, &ask)?;

    // burn msg
    let mut res = Response::new();
    if listing_fee > Uint128::zero() {
        fair_burn(listing_fee.u128(), None, &mut res);
    }

    let hook = prepare_ask_hook(deps.as_ref(), &ask, HookAction::Create)?;

    event = event
        .add_attribute("seller", seller)
        .add_attribute("price", price.to_string())
        .add_attribute("expires", expires.to_string());

    Ok(res.add_submessages(hook).add_event(event))
}
```

- 주문 실행

#### execute_remove_ask

```rust
pub fn execute_remove_ask(
    deps: DepsMut,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    only_owner(deps.as_ref(), &info, &collection, token_id)?;

    let key = ask_key(&collection, token_id);
    let ask = asks().load(deps.storage, key.clone())?;
    asks().remove(deps.storage, key)?;

    let hook = prepare_ask_hook(deps.as_ref(), &ask, HookAction::Delete)?;

    let event = Event::new("remove-ask")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string());

    Ok(Response::new().add_event(event).add_submessages(hook))
}
```

- 주문 삭제

#### execute_update_ask_price

```rust
pub fn execute_update_ask_price(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
    price: Coin,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    only_owner(deps.as_ref(), &info, &collection, token_id)?;
    price_validate(deps.storage, &price)?;

    let key = ask_key(&collection, token_id);

    let mut ask = asks().load(deps.storage, key.clone())?;
    if !ask.is_active {
        return Err(ContractError::AskNotActive {});
    }
    if ask.is_expired(&env.block) {
        return Err(ContractError::AskExpired {});
    }

    ask.price = price.amount;
    asks().save(deps.storage, key, &ask)?;

    let hook = prepare_ask_hook(deps.as_ref(), &ask, HookAction::Update)?;

    let event = Event::new("update-ask")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("price", price.to_string());

    Ok(Response::new().add_event(event).add_submessages(hook))
}
```

- 주문 가격 업데이트

#### execute_set_bid

```rust
pub fn execute_set_bid(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    sale_type: SaleType,
    bid_info: BidInfo,
    buy_now: bool,
) -> Result<Response, ContractError> {
    let BidInfo {
        collection,
        token_id,
        finders_fee_bps,
        expires,
        finder,
    } = bid_info;
    let params = SUDO_PARAMS.load(deps.storage)?;

    if let Some(finder) = finder.clone() {
        if info.sender == finder {
            return Err(ContractError::InvalidFinder(
                "bidder cannot be finder".to_string(),
            ));
        }
    }

    // check bid finders_fee_bps is not over max
    if let Some(fee) = finders_fee_bps {
        if Decimal::percent(fee) > params.max_finders_fee_percent {
            return Err(ContractError::InvalidFindersFeeBps(fee));
        }
    }
    let bid_price = must_pay(&info, NATIVE_DENOM)?;
    if bid_price < params.min_price {
        return Err(ContractError::PriceTooSmall(bid_price));
    }
    params.bid_expiry.is_valid(&env.block, expires)?;
    if let Some(finders_fee_bps) = finders_fee_bps {
        if Decimal::percent(finders_fee_bps) > params.max_finders_fee_percent {
            return Err(ContractError::InvalidFindersFeeBps(finders_fee_bps));
        }
    }

    let bidder = info.sender;
    let mut res = Response::new();
    let bid_key = bid_key(&collection, token_id, &bidder);
    let ask_key = ask_key(&collection, token_id);

    if let Some(existing_bid) = bids().may_load(deps.storage, bid_key.clone())? {
        bids().remove(deps.storage, bid_key)?;
        let refund_bidder = BankMsg::Send {
            to_address: bidder.to_string(),
            amount: vec![coin(existing_bid.price.u128(), NATIVE_DENOM)],
        };
        res = res.add_message(refund_bidder)
    }

    let existing_ask = asks().may_load(deps.storage, ask_key.clone())?;

    if let Some(ask) = existing_ask.clone() {
        if ask.is_expired(&env.block) {
            return Err(ContractError::AskExpired {});
        }
        if !ask.is_active {
            return Err(ContractError::AskNotActive {});
        }
        if let Some(reserved_for) = ask.reserve_for {
            if reserved_for != bidder {
                return Err(ContractError::TokenReserved {});
            }
        }
    } else if buy_now {
        return Err(ContractError::ItemNotForSale {});
    }

    let save_bid = |store| -> StdResult<_> {
        let bid = Bid::new(
            collection.clone(),
            token_id,
            bidder.clone(),
            bid_price,
            finders_fee_bps,
            expires,
        );
        store_bid(store, &bid)?;
        Ok(Some(bid))
    };

    let bid = match existing_ask {
        Some(ask) => match ask.sale_type {
            SaleType::FixedPrice => {
                // check if bid matches ask price then execute the sale
                // if the bid is lower than the ask price save the bid
                // otherwise return an error
                match bid_price.cmp(&ask.price) {
                    Ordering::Greater => {
                        return Err(ContractError::InvalidPrice {});
                    }
                    Ordering::Less => save_bid(deps.storage)?,
                    Ordering::Equal => {
                        asks().remove(deps.storage, ask_key)?;
                        let owner = match Cw721Contract::<Empty, Empty>(
                            ask.collection.clone(),
                            PhantomData,
                            PhantomData,
                        )
                        .owner_of(
                            &deps.querier,
                            ask.token_id.to_string(),
                            false,
                        ) {
                            Ok(res) => res.owner,
                            Err(_) => return Err(ContractError::InvalidListing {}),
                        };
                        if ask.seller != owner {
                            return Err(ContractError::InvalidListing {});
                        }
                        finalize_sale(
                            deps.as_ref(),
                            ask,
                            bid_price,
                            bidder.clone(),
                            finder,
                            &mut res,
                        )?;
                        None
                    }
                }
            }
            SaleType::Auction => {
                // check if bid price is equal or greater than ask price then place the bid
                // otherwise return an error
                match bid_price.cmp(&ask.price) {
                    Ordering::Greater => save_bid(deps.storage)?,
                    Ordering::Equal => save_bid(deps.storage)?,
                    Ordering::Less => {
                        return Err(ContractError::InvalidPrice {});
                    }
                }
            }
        },
        None => save_bid(deps.storage)?,
    };

    let hook = if let Some(bid) = bid {
        prepare_bid_hook(deps.as_ref(), &bid, HookAction::Create)?
    } else {
        vec![]
    };

    let event = Event::new("set-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("sale_type", sale_type.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("bidder", bidder)
        .add_attribute("bid_price", bid_price.to_string())
        .add_attribute("expires", expires.to_string());

    Ok(res.add_submessages(hook).add_event(event))
}
```

- 입찰 실행

#### execute_remove_bid

```rust
pub fn execute_remove_bid(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    let bidder = info.sender;

    let key = bid_key(&collection, token_id, &bidder);
    let bid = bids().load(deps.storage, key.clone())?;
    bids().remove(deps.storage, key)?;

    let refund_bidder_msg = BankMsg::Send {
        to_address: bid.bidder.to_string(),
        amount: vec![coin(bid.price.u128(), NATIVE_DENOM)],
    };

    let hook = prepare_bid_hook(deps.as_ref(), &bid, HookAction::Delete)?;

    let event = Event::new("remove-bid")
        .add_attribute("collection", collection)
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("bidder", bidder);

    let res = Response::new()
        .add_message(refund_bidder_msg)
        .add_event(event)
        .add_submessages(hook);

    Ok(res)
}
```

- 입찰 삭제

#### execute_accept_bid

```rust
pub fn execute_accept_bid(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
    bidder: Addr,
    finder: Option<Addr>,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    only_owner(deps.as_ref(), &info, &collection, token_id)?;
    only_tradable(deps.as_ref(), &env.block, &collection)?;
    let bid_key = bid_key(&collection, token_id, &bidder);
    let ask_key = ask_key(&collection, token_id);

    let bid = bids().load(deps.storage, bid_key.clone())?;
    if bid.is_expired(&env.block) {
        return Err(ContractError::BidExpired {});
    }

    if asks().may_load(deps.storage, ask_key.clone())?.is_some() {
        asks().remove(deps.storage, ask_key)?;
    }

    // Create a temporary Ask
    let ask = Ask {
        sale_type: SaleType::Auction,
        collection: collection.clone(),
        token_id,
        price: bid.price,
        expires_at: bid.expires_at,
        is_active: true,
        seller: info.sender.clone(),
        funds_recipient: Some(info.sender),
        reserve_for: None,
        finders_fee_bps: bid.finders_fee_bps,
    };

    // Remove accepted bid
    bids().remove(deps.storage, bid_key)?;

    let mut res = Response::new();

    // Transfer funds and NFT
    finalize_sale(
        deps.as_ref(),
        ask,
        bid.price,
        bidder.clone(),
        finder,
        &mut res,
    )?;

    let event = Event::new("accept-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("bidder", bidder)
        .add_attribute("price", bid.price.to_string());

    Ok(res.add_event(event))
}
```

- 입찰 수락

#### execute_reject_bid

```rust
pub fn execute_reject_bid(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
    bidder: Addr,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    only_owner(deps.as_ref(), &info, &collection, token_id)?;

    let bid_key = bid_key(&collection, token_id, &bidder);

    let bid = bids().load(deps.storage, bid_key.clone())?;
    if bid.is_expired(&env.block) {
        return Err(ContractError::BidExpired {});
    }

    bids().remove(deps.storage, bid_key)?;

    let refund_msg = BankMsg::Send {
        to_address: bidder.to_string(),
        amount: vec![coin(bid.price.u128(), NATIVE_DENOM.to_string())],
    };

    let hook = prepare_bid_hook(deps.as_ref(), &bid, HookAction::Delete)?;

    let event = Event::new("reject-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("bidder", bidder)
        .add_attribute("price", bid.price.to_string());

    Ok(Response::new()
        .add_event(event)
        .add_message(refund_msg)
        .add_submessages(hook))
}
```

- 입찰 거부

#### execute_set_collection_bid

```rust
pub fn execute_set_collection_bid(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    finders_fee_bps: Option<u64>,
    expires: Timestamp,
) -> Result<Response, ContractError> {
    let params = SUDO_PARAMS.load(deps.storage)?;
    let price = must_pay(&info, NATIVE_DENOM)?;
    if price < params.min_price {
        return Err(ContractError::PriceTooSmall(price));
    }
    params.bid_expiry.is_valid(&env.block, expires)?;
    // check bid finders_fee_bps is not over max
    if let Some(fee) = finders_fee_bps {
        if Decimal::percent(fee) > params.max_finders_fee_percent {
            return Err(ContractError::InvalidFindersFeeBps(fee));
        }
    }

    let bidder = info.sender;
    let mut res = Response::new();

    let key = collection_bid_key(&collection, &bidder);

    let existing_bid = collection_bids().may_load(deps.storage, key.clone())?;
    if let Some(bid) = existing_bid {
        collection_bids().remove(deps.storage, key.clone())?;
        let refund_bidder_msg = BankMsg::Send {
            to_address: bid.bidder.to_string(),
            amount: vec![coin(bid.price.u128(), NATIVE_DENOM)],
        };
        res = res.add_message(refund_bidder_msg);
    }

    let collection_bid = CollectionBid {
        collection: collection.clone(),
        bidder: bidder.clone(),
        price,
        finders_fee_bps,
        expires_at: expires,
    };
    collection_bids().save(deps.storage, key, &collection_bid)?;

    let hook = prepare_collection_bid_hook(deps.as_ref(), &collection_bid, HookAction::Create)?;

    let event = Event::new("set-collection-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("bidder", bidder)
        .add_attribute("bid_price", price.to_string())
        .add_attribute("expires", expires.to_string());

    Ok(res.add_event(event).add_submessages(hook))
}
```

- 컬렉션 입찰

#### execute_remove_collection_bid

```rust
pub fn execute_remove_collection_bid(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    collection: Addr,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    let bidder = info.sender;

    let key = collection_bid_key(&collection, &bidder);

    let collection_bid = collection_bids().load(deps.storage, key.clone())?;
    collection_bids().remove(deps.storage, key)?;

    let refund_bidder_msg = BankMsg::Send {
        to_address: collection_bid.bidder.to_string(),
        amount: vec![coin(collection_bid.price.u128(), NATIVE_DENOM)],
    };

    let hook = prepare_collection_bid_hook(deps.as_ref(), &collection_bid, HookAction::Delete)?;

    let event = Event::new("remove-collection-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("bidder", bidder);

    let res = Response::new()
        .add_message(refund_bidder_msg)
        .add_event(event)
        .add_submessages(hook);

    Ok(res)
}
```

- 컬렉션 입찰 삭제

#### execute_accept_collection_bid

```rust
pub fn execute_accept_collection_bid(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
    bidder: Addr,
    finder: Option<Addr>,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    only_owner(deps.as_ref(), &info, &collection, token_id)?;
    only_tradable(deps.as_ref(), &env.block, &collection)?;
    let bid_key = collection_bid_key(&collection, &bidder);
    let ask_key = ask_key(&collection, token_id);

    let bid = collection_bids().load(deps.storage, bid_key.clone())?;
    if bid.is_expired(&env.block) {
        return Err(ContractError::BidExpired {});
    }
    collection_bids().remove(deps.storage, bid_key)?;

    if asks().may_load(deps.storage, ask_key.clone())?.is_some() {
        asks().remove(deps.storage, ask_key)?;
    }

    // Create a temporary Ask
    let ask = Ask {
        sale_type: SaleType::Auction,
        collection: collection.clone(),
        token_id,
        price: bid.price,
        expires_at: bid.expires_at,
        is_active: true,
        seller: info.sender.clone(),
        funds_recipient: Some(info.sender.clone()),
        reserve_for: None,
        finders_fee_bps: bid.finders_fee_bps,
    };

    let mut res = Response::new();

    // Transfer funds and NFT
    finalize_sale(
        deps.as_ref(),
        ask,
        bid.price,
        bidder.clone(),
        finder,
        &mut res,
    )?;

    let event = Event::new("accept-collection-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("bidder", bidder)
        .add_attribute("seller", info.sender.to_string())
        .add_attribute("price", bid.price.to_string());

    Ok(res.add_event(event))
}
```

- 컬렉션 입찰 수락

#### execute_sync_ask

```rust
pub fn execute_sync_ask(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    only_operator(deps.storage, &info)?;

    let key = ask_key(&collection, token_id);

    let mut ask = asks().load(deps.storage, key.clone())?;

    // Check if marketplace still holds approval
    // An approval will be removed when
    // 1 - There is a transfer
    // 2 - The approval expired (approvals can have different expiration times)
    let res = Cw721Contract::<Empty, Empty>(collection.clone(), PhantomData, PhantomData).approval(
        &deps.querier,
        token_id.to_string(),
        env.contract.address.to_string(),
        None,
    );
    if res.is_ok() == ask.is_active {
        return Err(ContractError::AskUnchanged {});
    }
    ask.is_active = res.is_ok();
    asks().save(deps.storage, key, &ask)?;

    let hook = prepare_ask_hook(deps.as_ref(), &ask, HookAction::Update)?;

    let event = Event::new("update-ask-state")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("is_active", ask.is_active.to_string());

    Ok(Response::new().add_event(event).add_submessages(hook))
}
```

- 토큰 소유권 기준으로 요청 활성 상태 동기화

#### execute_remove_stale_ask

```rust
pub fn execute_remove_stale_ask(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    only_operator(deps.storage, &info)?;

    let key = ask_key(&collection, token_id);
    let ask = asks().load(deps.storage, key.clone())?;

    let res = Cw721Contract::<Empty, Empty>(collection.clone(), PhantomData, PhantomData).owner_of(
        &deps.querier,
        token_id.to_string(),
        false,
    );
    let has_owner = res.is_ok();
    let expired = ask.is_expired(&env.block);
    let mut has_approval = false;
    // Check if marketplace still holds approval for the collection/token_id
    // A CW721 approval will be removed when
    // 1 - There is a transfer or burn
    // 2 - The approval expired (CW721 approvals can have different expiration times)
    let res = Cw721Contract::<Empty, Empty>(collection.clone(), PhantomData, PhantomData).owner_of(
        &deps.querier,
        token_id.to_string(),
        false,
    );

    if res.is_ok() {
        has_approval = true;
    }

    // it has an owner and ask is still valid
    if has_owner && has_approval && !expired {
        return Err(ContractError::AskUnchanged {});
    }

    asks().remove(deps.storage, key)?;
    let hook = prepare_ask_hook(deps.as_ref(), &ask, HookAction::Delete)?;

    let event = Event::new("remove-ask")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("operator", info.sender.to_string())
        .add_attribute("expired", expired.to_string())
        .add_attribute("has_owner", has_owner.to_string())
        .add_attribute("has_approval", has_approval.to_string());

    Ok(Response::new().add_event(event).add_submessages(hook))
}
```

- 만료되거나 토큰이 삭제된 ask 삭제

#### execute_remove_stale_bid

```rust
pub fn execute_remove_stale_bid(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    token_id: TokenId,
    bidder: Addr,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    let operator = only_operator(deps.storage, &info)?;

    let bid_key = bid_key(&collection, token_id, &bidder);
    let bid = bids().load(deps.storage, bid_key.clone())?;

    let params = SUDO_PARAMS.load(deps.storage)?;
    let stale_time = (Expiration::AtTime(bid.expires_at) + params.stale_bid_duration)?;
    if !stale_time.is_expired(&env.block) {
        return Err(ContractError::BidNotStale {});
    }

    // bid is stale, refund bidder and reward operator
    bids().remove(deps.storage, bid_key)?;

    let reward = bid.price * params.bid_removal_reward_percent / Uint128::from(100u128);

    let bidder_msg = BankMsg::Send {
        to_address: bid.bidder.to_string(),
        amount: vec![coin((bid.price - reward).u128(), NATIVE_DENOM)],
    };
    let operator_msg = BankMsg::Send {
        to_address: operator.to_string(),
        amount: vec![coin(reward.u128(), NATIVE_DENOM)],
    };

    let hook = prepare_bid_hook(deps.as_ref(), &bid, HookAction::Delete)?;

    let event = Event::new("remove-stale-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("token_id", token_id.to_string())
        .add_attribute("bidder", bidder.to_string())
        .add_attribute("operator", operator.to_string())
        .add_attribute("reward", reward.to_string());

    Ok(Response::new()
        .add_event(event)
        .add_message(bidder_msg)
        .add_message(operator_msg)
        .add_submessages(hook))
}
```

- 만료되거나 토큰이 삭제된 입찰 제거

#### execute_remove_stale_collection_bid

```rust
pub fn execute_remove_stale_collection_bid(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    collection: Addr,
    bidder: Addr,
) -> Result<Response, ContractError> {
    nonpayable(&info)?;
    let operator = only_operator(deps.storage, &info)?;

    let key = collection_bid_key(&collection, &bidder);
    let collection_bid = collection_bids().load(deps.storage, key.clone())?;

    let params = SUDO_PARAMS.load(deps.storage)?;
    let stale_time = (Expiration::AtTime(collection_bid.expires_at) + params.stale_bid_duration)?;
    if !stale_time.is_expired(&env.block) {
        return Err(ContractError::BidNotStale {});
    }

    // collection bid is stale, refund bidder and reward operator
    collection_bids().remove(deps.storage, key)?;

    let reward = collection_bid.price * params.bid_removal_reward_percent / Uint128::from(100u128);

    let bidder_msg = BankMsg::Send {
        to_address: collection_bid.bidder.to_string(),
        amount: vec![coin((collection_bid.price - reward).u128(), NATIVE_DENOM)],
    };
    let operator_msg = BankMsg::Send {
        to_address: operator.to_string(),
        amount: vec![coin(reward.u128(), NATIVE_DENOM)],
    };

    let hook = prepare_collection_bid_hook(deps.as_ref(), &collection_bid, HookAction::Delete)?;

    let event = Event::new("remove-stale-collection-bid")
        .add_attribute("collection", collection.to_string())
        .add_attribute("bidder", bidder.to_string())
        .add_attribute("operator", operator.to_string())
        .add_attribute("reward", reward.to_string());

    Ok(Response::new()
        .add_event(event)
        .add_message(bidder_msg)
        .add_message(operator_msg)
        .add_submessages(hook))
}
```

- 만료되거나 토큰이 삭제된 컬렉션 입찰 삭제

#### finalize_sale

```rust
fn finalize_sale(
    deps: Deps,
    ask: Ask,
    price: Uint128,
    buyer: Addr,
    finder: Option<Addr>,
    res: &mut Response,
) -> StdResult<()> {
    payout(
        deps,
        ask.collection.clone(),
        price,
        ask.funds_recipient
            .clone()
            .unwrap_or_else(|| ask.seller.clone()),
        finder,
        ask.finders_fee_bps,
        res,
    )?;

    let cw721_transfer_msg = Cw721ExecuteMsg::TransferNft {
        token_id: ask.token_id.to_string(),
        recipient: buyer.to_string(),
    };

    let exec_cw721_transfer = WasmMsg::Execute {
        contract_addr: ask.collection.to_string(),
        msg: to_binary(&cw721_transfer_msg)?,
        funds: vec![],
    };
    res.messages.push(SubMsg::new(exec_cw721_transfer));

    res.messages
        .append(&mut prepare_sale_hook(deps, &ask, buyer.clone())?);

    let event = Event::new("finalize-sale")
        .add_attribute("collection", ask.collection.to_string())
        .add_attribute("token_id", ask.token_id.to_string())
        .add_attribute("seller", ask.seller.to_string())
        .add_attribute("buyer", buyer.to_string())
        .add_attribute("price", price.to_string());
    res.events.push(event);

    Ok(())
}
```

- 판매 종료 - 자금, NFT, 상태 변경

#### payout

```rust
fn payout(
    deps: Deps,
    collection: Addr,
    payment: Uint128,
    payment_recipient: Addr,
    finder: Option<Addr>,
    finders_fee_bps: Option<u64>,
    res: &mut Response,
) -> StdResult<()> {
    let params = SUDO_PARAMS.load(deps.storage)?;

    // Append Fair Burn message
    let network_fee = payment * params.trading_fee_percent / Uint128::from(100u128);
    fair_burn(network_fee.u128(), None, res);

    let collection_info: CollectionInfoResponse = deps
        .querier
        .query_wasm_smart(collection.clone(), &Sg721QueryMsg::CollectionInfo {})?;

    let finders_fee = match finder {
        Some(finder) => {
            let finders_fee = finders_fee_bps
                .map(|fee| (payment * Decimal::percent(fee) / Uint128::from(100u128)).u128())
                .unwrap_or(0);
            if finders_fee > 0 {
                res.messages.push(SubMsg::new(BankMsg::Send {
                    to_address: finder.to_string(),
                    amount: vec![coin(finders_fee, NATIVE_DENOM)],
                }));
            }
            finders_fee
        }
        None => 0,
    };

    match parse_royalties(collection_info.royalty_info) {
        // If token supports royalties, payout shares to royalty recipient
        Some(royalty) => {
            let amount = coin((payment * royalty.share).u128(), NATIVE_DENOM);
            if payment < (network_fee + Uint128::from(finders_fee) + amount.amount) {
                return Err(StdError::generic_err("Fees exceed payment"));
            }
            res.messages.push(SubMsg::new(BankMsg::Send {
                to_address: royalty.payment_address.to_string(),
                amount: vec![amount.clone()],
            }));
            let event = Event::new("royalty-payout")
                .add_attribute("collection", collection.to_string())
                .add_attribute("amount", amount.to_string())
                .add_attribute("recipient", royalty.payment_address.to_string());
            res.events.push(event);
            let seller_share_msg = BankMsg::Send {
                to_address: payment_recipient.to_string(),
                amount: vec![coin(
                    (payment * (Decimal::one() - royalty.share) - network_fee).u128() - finders_fee,
                    NATIVE_DENOM.to_string(),
                )],
            };
            res.messages.push(SubMsg::new(seller_share_msg));
        }
        None => {
            if payment < (network_fee + Uint128::from(finders_fee)) {
                return Err(StdError::generic_err("Fees exceed payment"));
            }
            // If token doesn't support royalties, pay seller in full
            let seller_share_msg = BankMsg::Send {
                to_address: payment_recipient.to_string(),
                amount: vec![coin(
                    (payment - network_fee).u128() - finders_fee,
                    NATIVE_DENOM.to_string(),
                )],
            };
            res.messages.push(SubMsg::new(seller_share_msg));
        }
    }

    Ok(())
}
```

- 입찰의 가격 지불

### sudo.rs

#### sudoParam

```rust
const MAX_FEE_BPS: u64 = 10000;

pub struct ParamInfo {
    trading_fee_bps: Option<u64>,
    ask_expiry: Option<ExpiryRange>,
    bid_expiry: Option<ExpiryRange>,
    operators: Option<Vec<String>>,
    max_finders_fee_bps: Option<u64>,
    min_price: Option<Uint128>,
    stale_bid_duration: Option<u64>,
    bid_removal_reward_bps: Option<u64>,
    listing_fee: Option<Uint128>,
}
```

#### sudo_update_params

```rust
/// Only governance can update contract params
pub fn sudo_update_params(
    deps: DepsMut,
    _env: Env,
    param_info: ParamInfo,
) -> Result<Response, ContractError> {
    let ParamInfo {
        trading_fee_bps,
        ask_expiry,
        bid_expiry,
        operators: _operators,
        max_finders_fee_bps,
        min_price,
        stale_bid_duration,
        bid_removal_reward_bps,
        listing_fee,
    } = param_info;
    if let Some(max_finders_fee_bps) = max_finders_fee_bps {
        if max_finders_fee_bps > MAX_FEE_BPS {
            return Err(ContractError::InvalidFindersFeeBps(max_finders_fee_bps));
        }
    }
    if let Some(trading_fee_bps) = trading_fee_bps {
        if trading_fee_bps > MAX_FEE_BPS {
            return Err(ContractError::InvalidTradingFeeBps(trading_fee_bps));
        }
    }
    if let Some(bid_removal_reward_bps) = bid_removal_reward_bps {
        if bid_removal_reward_bps > MAX_FEE_BPS {
            return Err(ContractError::InvalidBidRemovalRewardBps(
                bid_removal_reward_bps,
            ));
        }
    }

    ask_expiry.as_ref().map(|a| a.validate()).transpose()?;
    bid_expiry.as_ref().map(|b| b.validate()).transpose()?;

    let mut params = SUDO_PARAMS.load(deps.storage)?;

    params.trading_fee_percent = trading_fee_bps
        .map(Decimal::percent)
        .unwrap_or(params.trading_fee_percent);

    params.ask_expiry = ask_expiry.unwrap_or(params.ask_expiry);
    params.bid_expiry = bid_expiry.unwrap_or(params.bid_expiry);

    params.max_finders_fee_percent = max_finders_fee_bps
        .map(Decimal::percent)
        .unwrap_or(params.max_finders_fee_percent);

    params.min_price = min_price.unwrap_or(params.min_price);

    params.stale_bid_duration = stale_bid_duration
        .map(Duration::Time)
        .unwrap_or(params.stale_bid_duration);

    params.bid_removal_reward_percent = bid_removal_reward_bps
        .map(Decimal::percent)
        .unwrap_or(params.bid_removal_reward_percent);

    params.listing_fee = listing_fee.unwrap_or(params.listing_fee);

    SUDO_PARAMS.save(deps.storage, &params)?;

    Ok(Response::new().add_attribute("action", "update_params"))
}
```

- 설정 수정

#### sudo_add_operator

```rust
pub fn sudo_add_operator(deps: DepsMut, operator: Addr) -> Result<Response, ContractError> {
    let mut params = SUDO_PARAMS.load(deps.storage)?;
    if !params.operators.iter().any(|o| o == &operator) {
        params.operators.push(operator.clone());
    } else {
        return Err(ContractError::OperatorAlreadyRegistered {});
    }
    SUDO_PARAMS.save(deps.storage, &params)?;
    let res = Response::new()
        .add_attribute("action", "add_operator")
        .add_attribute("operator", operator);
    Ok(res)
}
```

- 오퍼레이터 추가

#### sudo_remove_operator

```rust
pub fn sudo_remove_operator(deps: DepsMut, operator: Addr) -> Result<Response, ContractError> {
    let mut params = SUDO_PARAMS.load(deps.storage)?;
    if let Some(i) = params.operators.iter().position(|o| o == &operator) {
        params.operators.remove(i);
    } else {
        return Err(ContractError::OperatorNotRegistered {});
    }
    SUDO_PARAMS.save(deps.storage, &params)?;
    let res = Response::new()
        .add_attribute("action", "remove_operator")
        .add_attribute("operator", operator);
    Ok(res)
}
```

- 오퍼레이터 삭제

### helpers.rs

#### MarketplaceContract

```rust
#[cw_serde]
pub struct MarketplaceContract(pub Addr);

impl MarketplaceContract {
    pub fn addr(&self) -> Addr {
        self.0.clone()
    }

    pub fn call<T: Into<ExecuteMsg>>(&self, msg: T) -> StdResult<CosmosMsg> {
        let msg = to_binary(&msg.into())?;
        Ok(WasmMsg::Execute {
            contract_addr: self.addr().into(),
            msg,
            funds: vec![],
        }
        .into())
    }
}
```

- 마켓 컨트랙트 주소를 감싸는 wrapper

#### map_validate

```rust
pub fn map_validate(api: &dyn Api, addresses: &[String]) -> StdResult<Vec<Addr>> {
    let mut validated_addresses = addresses
        .iter()
        .map(|addr| api.addr_validate(addr))
        .collect::<StdResult<Vec<_>>>()?;
    validated_addresses.sort();
    validated_addresses.dedup();
    Ok(validated_addresses)
}
```

- address 배열 검증

#### ExpiryRange

```rust
#[derive(Error, Debug, PartialEq)]
pub enum ExpiryRangeError {
    #[error("{0}")]
    Std(#[from] StdError),

    #[error("Invalid expiration range")]
    InvalidExpirationRange {},

    #[error("Expiry min > max")]
    InvalidExpiry {},
}

#[cw_serde]
pub struct ExpiryRange {
    pub min: u64,
    pub max: u64,
}

impl ExpiryRange {
    pub fn new(min: u64, max: u64) -> Self {
        ExpiryRange { min, max }
    }

    /// Validates if given expires time is within the allowable range
    pub fn is_valid(&self, block: &BlockInfo, expires: Timestamp) -> Result<(), ExpiryRangeError> {
        let now = block.time;
        if !(expires > now.plus_seconds(self.min) && expires <= now.plus_seconds(self.max)) {
            return Err(ExpiryRangeError::InvalidExpirationRange {});
        }

        Ok(())
    }

    pub fn validate(&self) -> Result<(), ExpiryRangeError> {
        if self.min > self.max {
            return Err(ExpiryRangeError::InvalidExpiry {});
        }

        Ok(())
    }
}
```

- 만료 시간 범위 검증

## 참고

[bps](https://ko.wikipedia.org/wiki/BPS)
