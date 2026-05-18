---
layout: post
title: "K8s Deep Week 8: Operator 개발 — Kubebuilder로 CRD + Controller 처음부터"
date: 2026-05-18 15:00:00 +0900
categories: kubernetes platform devops senior-series operator
---

## 학습 목표

- "Operator 패턴" 이 K8s 확장의 핵심인 이유를 안다.
- Kubebuilder로 CRD + Controller 스캐폴딩.
- Reconciler 함수에서 desired/actual 비교를 직접 구현.
- 운영 가능한 controller로 만드는 패턴(finalizer, owner reference, event)을 안다.

## 1. 비유 — "도메인 전문가를 K8s 안에 채용"

K8s 기본 객체(Deployment, Service)는 일반화된 도구입니다. 그런데 "MySQL 클러스터" 같은 도메인 전문 지식은 K8s가 모릅니다.

**Operator** = 그 도메인의 전문가(예: DBA)의 운영 지식을 K8s controller로 코드화한 것. CR(Custom Resource)을 만들면 controller가 그 도메인 작업(primary 승격, replica 추가, 백업)을 자동 수행.

대표 Operator: PostgreSQL Operator (Zalando, CrunchyData), Elasticsearch Operator (Elastic), Prometheus Operator, ArgoCD Operator.

## 2. Operator = CRD + Controller

- **CRD (Custom Resource Definition)**: 새 K8s API 객체 정의. "MyDatabase" 라는 새 타입을 알려주는 스키마.
- **Custom Resource (CR)**: 그 타입의 인스턴스. `kind: MyDatabase`.
- **Controller**: CR을 watch하며 reconciliation 수행하는 프로그램.

## 3. Kubebuilder 스캐폴딩

```bash
mkdir my-operator && cd my-operator
kubebuilder init --domain=mycorp.com --repo=github.com/mycorp/my-operator
kubebuilder create api --group=db --version=v1 --kind=MyDatabase
# 자동으로 api/v1/mydatabase_types.go, controllers/mydatabase_controller.go 생성
```

생성된 디렉토리:
```
api/v1/
  mydatabase_types.go         # CRD 스키마 (Go struct)
controllers/
  mydatabase_controller.go    # Reconcile 함수
config/
  crd/                         # 생성된 CRD YAML
  rbac/
  manager/
main.go                        # operator 시작점
```

## 4. CRD 스키마 작성

`api/v1/mydatabase_types.go`:

```go
type MyDatabaseSpec struct {
    // 원하는 replica 수
    Replicas int32 `json:"replicas"`

    // MySQL 버전
    Version string `json:"version"`

    // 백업 활성?
    BackupEnabled bool `json:"backupEnabled,omitempty"`
}

type MyDatabaseStatus struct {
    // 현재 phase
    Phase string `json:"phase,omitempty"`

    // ready replica
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`

    // 마지막 백업 시각
    LastBackupTime *metav1.Time `json:"lastBackupTime,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type MyDatabase struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   MyDatabaseSpec   `json:"spec,omitempty"`
    Status MyDatabaseStatus `json:"status,omitempty"`
}
```

`make manifests` → config/crd 안에 CRD YAML 자동 생성.

## 5. Reconciler 작성

`controllers/mydatabase_controller.go`:

```go
func (r *MyDatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 1. CR 로드
    var db dbv1.MyDatabase
    if err := r.Get(ctx, req.NamespacedName, &db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. desired StatefulSet
    desired := buildStatefulSet(&db)

    // 3. actual StatefulSet
    var actual appsv1.StatefulSet
    err := r.Get(ctx, client.ObjectKey{Namespace: db.Namespace, Name: db.Name}, &actual)
    if errors.IsNotFound(err) {
        // 4. 없으면 생성
        ctrl.SetControllerReference(&db, desired, r.Scheme)
        if err := r.Create(ctx, desired); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("StatefulSet created")
        return ctrl.Result{Requeue: true}, nil
    } else if err != nil {
        return ctrl.Result{}, err
    }

    // 5. 차이 비교 & 업데이트
    if *actual.Spec.Replicas != db.Spec.Replicas {
        actual.Spec.Replicas = &db.Spec.Replicas
        if err := r.Update(ctx, &actual); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 6. status 업데이트
    db.Status.ReadyReplicas = actual.Status.ReadyReplicas
    db.Status.Phase = "Running"
    if err := r.Status().Update(ctx, &db); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *MyDatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&dbv1.MyDatabase{}).
        Owns(&appsv1.StatefulSet{}).
        Complete(r)
}
```

## 6. 필수 패턴 4종

### (a) Owner Reference
`ctrl.SetControllerReference(parent, child, scheme)` — 부모 삭제 시 자식도 자동 삭제 (cascading delete).

### (b) Finalizer
삭제 직전에 외부 정리(예: S3 백업 폴더 비우기) 필요할 때.

```go
const finalizerName = "mydatabase.mycorp.com/finalizer"

if db.DeletionTimestamp.IsZero() {
    if !containsString(db.Finalizers, finalizerName) {
        db.Finalizers = append(db.Finalizers, finalizerName)
        r.Update(ctx, &db)
    }
} else {
    if containsString(db.Finalizers, finalizerName) {
        // 외부 정리
        if err := cleanupS3(&db); err != nil { return ctrl.Result{}, err }
        db.Finalizers = removeString(db.Finalizers, finalizerName)
        r.Update(ctx, &db)
    }
    return ctrl.Result{}, nil
}
```

### (c) Event 기록
`r.Recorder.Event(&db, "Normal", "Created", "StatefulSet created")` — `kubectl describe` 에 표시.

### (d) Status Condition
phase 단일 string 대신 conditions 배열 (Ready/Progressing/Degraded 등). API convention.

## 7. RBAC 자동 생성

Reconciler 위에 marker comment:

```go
// +kubebuilder:rbac:groups=db.mycorp.com,resources=mydatabases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=db.mycorp.com,resources=mydatabases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
```

`make manifests` 가 ClusterRole 자동 생성.

## 8. 빌드 & 배포

```bash
make docker-build IMG=mycorp/my-operator:v0.1
make docker-push IMG=mycorp/my-operator:v0.1
make deploy IMG=mycorp/my-operator:v0.1

# CR 만들기
cat <<EOF | kubectl apply -f -
apiVersion: db.mycorp.com/v1
kind: MyDatabase
metadata: { name: my-db }
spec:
  replicas: 3
  version: "8.0"
  backupEnabled: true
EOF

# StatefulSet 자동 생성 확인
kubectl get statefulset my-db
```

## 9. 운영 함정 5선

1. **무한 reconcile loop**: status 업데이트가 다시 reconcile 트리거 → 무한. resourceVersion + 변경 검사로 차단.
2. **status를 spec 으로 업데이트**: spec 변경은 다시 reconcile → loop.
3. **finalizer 빠뜨려 외부 자원 누수**: 삭제 후 S3 폴더 잔존.
4. **RBAC 누락**: operator가 CRUD 권한 없어 동작 안 함.
5. **error wrap 누락**: 디버깅 지옥.

## 10. 빅테크 사례 — Adobe의 자체 Operator

Adobe는 사내 PaaS 위에 도메인 특화 operator 수십 개. 예: "Tenant" CRD가 namespace + RBAC + quota + monitoring + alert을 한 번에 프로비저닝. 100+ 팀 셀프서비스가 가능.

## 11. 실습

위 코드를 그대로 따라 my-operator를 만들고:
1. MyDatabase CR 변경(replicas: 3 → 5) 시 StatefulSet도 갱신되는지.
2. CR 삭제 시 StatefulSet 자동 삭제 (owner reference).
3. 일부러 status 업데이트를 loop로 만들어 무한 reconcile 발생시킨 후 패치.
4. Event를 추가해 `kubectl describe`에서 확인.

## 12. 자가평가 퀴즈

### Q1. Operator의 본질?
1. **도메인 지식을 controller로 코드화 + CRD**
2. 단순 자동화
3. UI
4. 무관

**정답: 1.**

### Q2. Owner Reference 의 효과?
1. **부모 삭제 시 자식 자동 cascading delete**
2. UI 색
3. 비용
4. 무관

**정답: 1.**

### Q3. Finalizer 의 용도?
1. **삭제 직전 외부 정리(백업 폴더 등)**
2. 생성 직전
3. UI
4. 무관

**정답: 1.**

### Q4. 무한 reconcile 흔한 원인?
1. **status 변경이 다시 reconcile을 트리거**
2. UI
3. 무관
4. 정상

**정답: 1.**

### Q5. CRD에 subresource:status를 두는 이유?
1. **status update가 spec과 분리된 권한으로 처리 — controller만 status 수정**
2. UI
3. 비용
4. 무관

**정답: 1.**

## 13. 다음 주차

[Week 9: 보안]에서는 RBAC, PSA, NetworkPolicy, etcd encryption을 다룹니다.
