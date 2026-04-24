# T3-001_ERP_INTEGRATION_v1.md

## 1. 문서 목적
Phase 1(MVP) 범위에서는 ERP/MES 실시간 연동이 제외되고 Bulk Import(CSV)로 대체된다. 그러나 Phase 2 연동을 위한 인터페이스 추상화 설계와, MVP 단계에서의 CSV 기반 단방향 동기화 운영 절차를 완전히 정의한다.

## 2. Phase 1 vs Phase 2 연동 범위 명확화

| 항목 | Phase 1 (MVP) | Phase 2 |
|------|-------------|---------|
| 데이터 흐름 방향 | 단방향 (ERP → 시스템, CSV 통해) | 양방향 실시간 |
| 동기화 주기 | 수동 (관리자 업로드 시) | 자동 (Webhook 또는 폴링) |
| 지원 ERP 시스템 | 해당 없음 | SAP, Oracle ERP, 더존, 영림원, 그룹웨어 |
| 연동 방식 | CSV/Excel Bulk Import | REST API 또는 ERP SDK 직접 연동 |

## 3. Phase 1 CSV 기반 운영 절차

### 3.1. 정기 동기화 권장 주기
- **기준 정보 (제품/공정 마스터)**: 월 1회 또는 신제품/신공정 도입 시 즉시 갱신
- **BOM (자재명세서)**: 주 1회 또는 설계 변경 시점 (ECO 발행 시) 갱신

### 3.2. 관리자 운영 체크리스트
1. **업로드 전 검증**: ERP에서 추출된 CSV 파일의 인코딩(UTF-8) 및 필수 컬럼 누락 여부 사전 확인.
2. **업로드**: 어드민 대시보드의 Bulk Import 메뉴를 통해 파일 업로드 (미리보기 화면에서 파싱 결과 확인).
3. **오류 처리**: 데이터 포맷 오류(`VAL_400`) 발생 시 화면에 하이라이트된 라인을 수정 후 재업로드 (에러 무시 후 강제 업로드 불가 원칙).
4. **확인**: 업로드 완료 후 생성된 배치 작업 로그 및 최종 적용된 데이터 행 수 교차 검증.

### 3.3. 데이터 충돌 처리 정책
- **정책**: 기존 코드와 동일한 행 업로드 시 **버전 이력 저장 후 덮어쓰기 (Upsert & Archive)**
- **근거 및 구현 방법**: 
  - 품질 감사에서는 과거 시점의 제품 정보 추적이 필수적이므로 단순 덮어쓰기나 스킵은 데이터 무결성에 위배된다.
  - 데이터 식별자(예: `product_code`) 충돌 시, 기존 레코드의 `is_active` 플래그를 `false`로 변경(Soft Delete)하고 새로운 레코드를 `is_active=true`로 Insert하는 방식을 취한다. `audit_log`를 통해 변경 이력이 완벽히 보장된다.

## 4. Phase 2 대비 추상화 인터페이스 설계

Phase 2에서 실제 ERP API 연동 시 기존 로직을 최대한 유지하기 위해 데이터 소스를 추상화한다.

```typescript
// src/lib/integration/DataImportSource.ts

export interface ProductMaster {
  productCode: string;
  productName: string;
  specifications: Record<string, any>;
  isActive: boolean;
}

export interface ProcessMaster {
  processCode: string;
  processName: string;
  facilities: string[];
}

export interface BOMItem {
  parentCode: string;
  childCode: string;
  quantity: number;
}

// 데이터 소스 추상 인터페이스
export interface DataImportSource {
  fetchProducts(): Promise<ProductMaster[]>;
  fetchProcesses(): Promise<ProcessMaster[]>;
  fetchBOM(): Promise<BOMItem[]>;
  getLastSyncTimestamp(): Promise<Date>;
}

// Phase 1 구현체 (CSV 파일 파서)
export class CSVImportSource implements DataImportSource {
  constructor(private fileBuffer: Buffer) {}

  async fetchProducts(): Promise<ProductMaster[]> {
    // CSV 파싱 로직 구현 (PapaParse 등 활용)
    console.log("Parsing products from CSV...");
    return []; 
  }

  async fetchProcesses(): Promise<ProcessMaster[]> {
    console.log("Parsing processes from CSV...");
    return [];
  }

  async fetchBOM(): Promise<BOMItem[]> {
    console.log("Parsing BOM from CSV...");
    return [];
  }

  async getLastSyncTimestamp(): Promise<Date> {
    // 파일 업로드 시점 반환
    return new Date();
  }
}

// Phase 2 구현체 예시 (SAP RFC 연동 - 스텁 수준)
export class SAPImportSource implements DataImportSource {
  constructor(private sapClientConfig: any) {}

  async fetchProducts(): Promise<ProductMaster[]> {
    // SAP RFC 호출 로직
    console.log("Fetching products via SAP RFC...");
    return [];
  }

  async fetchProcesses(): Promise<ProcessMaster[]> {
    console.log("Fetching processes via SAP RFC...");
    return [];
  }

  async fetchBOM(): Promise<BOMItem[]> {
    console.log("Fetching BOM via SAP RFC...");
    return [];
  }

  async getLastSyncTimestamp(): Promise<Date> {
    // SAP 시스템 상의 최근 변경일 조회
    return new Date();
  }
}
```

## 5. 정부 혁신바우처 API 연동 명세

Project Context에 명시된 유일한 외부 API 연동 항목으로, 고객사가 바우처 지원 대상인지 확인한다.

### 5.1. API 엔드포인트 및 인증
- **대상**: 중소벤처기업진흥공단 혁신바우처 시스템 오픈 API (가정)
- **인증 방식**: 공공데이터포털 인증키(API Key)를 헤더 또는 Query 파라미터로 전달 (`Authorization: Bearer <API_KEY>`)

### 5.2. 조회 가능한 데이터 항목
- 바우처 자격 여부 (`is_eligible`: boolean)
- 사용 가능 한도액 (`available_limit`: number)
- 바우처 만료일 (`expiration_date`: string)
- 사용 이력 요약 (`usage_summary`: string)

### 5.3. Route Handler 구현 명세

```typescript
// app/api/admin/innovation-voucher/status/route.ts
import { NextResponse } from 'next/server';
import { handleRouteError } from '@/lib/errors';

export const runtime = 'edge';

// Vercel Edge Cache TTL 설정 (1시간 캐싱)
export const revalidate = 3600;

export async function GET(request: Request) {
  try {
    const { searchParams } = new URL(request.url);
    const companyBizNo = searchParams.get('bizNo');

    if (!companyBizNo) {
      throw new Error('VAL_400_INVALID_FIELD: 사업자등록번호가 필요합니다.');
    }

    const res = await fetch(`https://api.voucher.go.kr/v1/status?bizNo=${companyBizNo}`, {
      headers: {
        'Authorization': `Bearer ${process.env.VOUCHER_API_KEY}`,
      },
      // Next.js fetch 캐싱 (edge cache 적용)
      next: { revalidate: 3600 }
    });

    if (!res.ok) {
      throw new Error('EXT_502_EXTERNAL_SERVICE_ERROR: 바우처 API 응답 오류');
    }

    const data = await res.json();
    
    return NextResponse.json({
      success: true,
      data: {
        is_eligible: data.eligible,
        available_limit: data.limit,
        expiration_date: data.expireAt,
      }
    });
  } catch (error) {
    return handleRouteError(error);
  }
}
```

### 5.4. UI 표시 위치 및 방식
- **위치**: Admin 대시보드 메인 상단 배너 및 설정(Settings) > '결제 및 구독' 페이지
- **방식**: `is_eligible`이 `true`일 경우, "🎉 혁신바우처 지원 대상 기업입니다 (잔여 한도: ₩XX,XXX)" 형태의 강조된 알림 배너 노출.

## 6. 데이터 마이그레이션 안전 절차

대량의 기준 정보(마스터 데이터) 교체 시 데이터 정합성을 보호하기 위한 안전 장치를 마련한다.

### 6.1. 대량 마스터 데이터 교체 시 롤백 방안
1. **스냅샷 생성 (PITR)**: Supabase의 Point-in-Time Recovery 기능을 활용하여 대규모 업로드 직전에 스냅샷 시간을 기록한다. (Free Tier 제약 시 수동으로 대상 테이블의 데이터를 `archived_table`로 SQL 덤프 후 업로드 수행)
2. **롤백 절차**: 교체 실패(치명적인 데이터 오염 발견) 시, 관리자 패널의 '마이그레이션 롤백' 버튼을 통해 직전 트랜잭션을 무효화하거나 백업된 `archived_table` 데이터를 원복하는 복구 스크립트를 즉각 실행한다. `audit_log`를 역추적하여 방금 Insert된 `bulk_import_batches` ID에 해당하는 레코드들을 Soft Delete 처리한다.

### 6.2. 마이그레이션 중 서비스 무중단 방안
- **판단**: Serverless 환경(Vercel)과 Supabase 단일 인스턴스 체제에서는 완전한 인프라 수준의 Blue/Green 배포 구현이 비용 원칙($0)상 어렵다.
- **적용 방안 (논리적 Blue/Green)**: 데이터베이스 내부에 데이터 버저닝 방식을 적용한다. 
  1. 새 마스터 데이터 세트를 `version = N+1`로 백그라운드에서 Insert 한다. 
  2. Insert 완료 시 시스템 설정 테이블의 `active_master_version` 플래그를 `N+1`로 원자적으로(Atomic Update) 스위칭한다. 
  3. 이 과정에서 읽기 작업은 여전히 `version = N`을 조회하므로 **다운타임 없이** 마이그레이션과 조회가 동시에 가능하다.
