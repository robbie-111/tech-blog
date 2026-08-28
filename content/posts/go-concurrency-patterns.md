+++
title = "Go 동시성 패턴: goroutine 누수 없이 병렬 작업 다루기"
date = 2026-08-28T10:00:00+09:00
tags = ["go", "concurrency", "backend"]
categories = ["Go"]
summary = "WaitGroup부터 errgroup까지, 실무에서 반복해서 쓰게 되는 Go 동시성 패턴 다섯 가지와 그 과정에서 밟기 쉬운 함정들을 정리한다."
draft = false
+++

Go에서 `go` 키워드 하나로 goroutine을 띄우는 건 쉽다. 어려운 건 그 goroutine이 **언제 끝나는지**, 그리고 **끝나긴 하는지**를 보장하는 일이다.

이 글은 실무에서 반복해서 마주치는 다섯 가지 패턴을 정리한다. 각 패턴은 "무엇을 해결하는가"보다 "언제 부족해지는가"에 초점을 맞췄다.

## 1. WaitGroup: 시작한 만큼 기다린다

가장 기본. N개의 작업을 병렬로 던지고 전부 끝날 때까지 기다린다.

```go
func fetchAll(urls []string) []string {
	var wg sync.WaitGroup
	results := make([]string, len(urls))

	for i, url := range urls {
		wg.Add(1)
		go func() {
			defer wg.Done()
			results[i] = fetch(url)
		}()
	}

	wg.Wait()
	return results
}
```

각 goroutine이 `results`의 **서로 다른 인덱스**에만 쓰기 때문에 뮤텍스가 필요 없다. 슬라이스에 `append`를 했다면 이야기가 달라진다 — 그때는 락이나 채널이 필요하다.

`wg.Add(1)`은 반드시 `go` 앞에 와야 한다. goroutine 안에서 `Add`를 호출하면 `Wait`가 먼저 실행돼 그대로 통과해버릴 수 있다.

WaitGroup의 한계는 명확하다. **에러를 돌려받을 방법이 없고, 중간에 멈출 방법도 없다.** 100개 중 첫 번째가 실패해도 나머지 99개는 끝까지 돈다. 이 두 가지가 필요해지는 순간이 5번 `errgroup`으로 넘어갈 때다.

Go 1.25부터는 `wg.Go(func(){...})`로 `Add`/`Done` 쌍을 생략할 수 있다. 실수 여지가 줄어드니 새 코드라면 이쪽이 낫다.

## 2. 채널 소유권: 보내는 쪽이 닫는다

채널에서 나오는 버그의 상당수는 "누가 닫을 것인가"가 정해지지 않아서 생긴다. 규칙은 하나다.

> **채널을 만들고 보내는 쪽이 닫는다. 받는 쪽은 절대 닫지 않는다.**

이 규칙을 타입으로 강제할 수 있다. 방향성 채널을 쓰면 된다.

```go
// 반환 타입이 <-chan 이므로 호출자는 close()를 호출할 수 없다.
func generate(ctx context.Context, nums []int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out) // 소유자가 책임지고 닫는다
		for _, n := range nums {
			select {
			case out <- n:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}
```

두 가지가 중요하다.

- 반환 타입이 `<-chan int`라서 **호출자 쪽에서는 `close(out)`이 컴파일 에러**가 된다. 규칙을 주석이 아니라 타입 시스템에 새기는 것이다.
- 보내는 부분이 `select`로 감싸여 있다. 이게 없으면 소비자가 중간에 읽기를 그만뒀을 때 `out <- n`이 영원히 블록되고, 이 goroutine은 프로세스가 끝날 때까지 살아있는다. **전형적인 goroutine 누수다.**

닫힌 채널에서 읽으면 즉시 제로값이 나온다. 그래서 `for v := range ch`는 채널이 닫히면 자연스럽게 빠져나온다. 반대로 닫힌 채널에 보내면 panic이다. 이 비대칭이 "보내는 쪽이 닫는다" 규칙의 근거다.

## 3. select + context: 취소를 전파한다

`context.Context`는 Go에서 취소 신호를 전달하는 표준 통로다. 핵심은 `ctx.Done()`이 **채널**이라서 `select`에 그대로 얹을 수 있다는 점이다.

```go
func fetchWithTimeout(parent context.Context, url string) (string, error) {
	ctx, cancel := context.WithTimeout(parent, 2*time.Second)
	defer cancel() // 성공/실패와 무관하게 항상 호출한다

	resultCh := make(chan string, 1) // 버퍼 1이 핵심

	go func() {
		resultCh <- slowFetch(url)
	}()

	select {
	case res := <-resultCh:
		return res, nil
	case <-ctx.Done():
		return "", ctx.Err() // DeadlineExceeded 또는 Canceled
	}
}
```

여기서 놓치기 쉬운 두 줄이 있다.

**`defer cancel()`.** 호출하지 않으면 타이머가 만료될 때까지 context와 그에 딸린 리소스가 해제되지 않는다. `go vet`이 잡아주는 항목이기도 하다.

**`make(chan string, 1)`.** 버퍼가 없으면 타임아웃이 먼저 발생했을 때 `select`는 `ctx.Done()`으로 빠져나가고, 뒤늦게 끝난 goroutine은 `resultCh <-`에서 받는 사람 없이 영원히 블록된다. 버퍼 1칸이 있으면 값을 던져놓고 goroutine이 정상 종료한다. **버려질 결과라도 받아줄 자리는 남겨둬야 한다.**

`ctx.Err()`로 원인을 구분할 수 있다. `context.DeadlineExceeded`면 타임아웃, `context.Canceled`면 상위에서 취소된 것이다.

## 4. Worker Pool: 동시성에 상한을 둔다

입력이 10개면 goroutine 10개도 괜찮다. 10만 개라면? 외부 API를 10만 번 동시에 때리거나 DB 커넥션 풀을 말려버린다. 동시 실행 개수에 천장이 필요하다.

가장 단순한 방법은 **버퍼 채널을 세마포어로 쓰는 것**이다.

```go
func processAll(ctx context.Context, jobs []Job, limit int) error {
	sem := make(chan struct{}, limit) // limit 칸짜리 세마포어
	var wg sync.WaitGroup

	for _, job := range jobs {
		select {
		case sem <- struct{}{}: // 자리가 빌 때까지 대기
		case <-ctx.Done():
			wg.Wait()
			return ctx.Err()
		}

		wg.Add(1)
		go func() {
			defer wg.Done()
			defer func() { <-sem }() // 자리 반납
			process(job)
		}()
	}

	wg.Wait()
	return nil
}
```

`chan struct{}`를 쓰는 이유는 빈 구조체가 **0바이트**라서다. 신호만 필요하고 값은 필요 없을 때의 관용구다.

자리 획득도 `select`로 감쌌다. 이게 없으면 취소된 뒤에도 남은 작업을 계속 밀어넣게 된다.

작업 수가 고정적이고 워커를 재사용하고 싶다면 고전적인 형태가 더 맞는다. 워커 N개가 하나의 job 채널을 나눠 읽는 방식이다.

```go
func workerPool(jobs <-chan Job, workers int) {
	var wg sync.WaitGroup
	for range workers { // Go 1.22+: int 범위 순회
		wg.Add(1)
		go func() {
			defer wg.Done()
			for job := range jobs { // 채널이 닫히면 종료
				process(job)
			}
		}()
	}
	wg.Wait()
}
```

이 형태의 장점은 goroutine이 작업 수가 아니라 `workers` 수만큼만 생성된다는 것이다. 종료 조건은 전적으로 **생산자가 `jobs`를 닫는 것**에 달려 있다. 안 닫으면 `wg.Wait()`에서 영원히 멈춘다.

## 5. errgroup: 에러 수집과 취소를 한꺼번에

`golang.org/x/sync/errgroup`은 WaitGroup의 두 가지 한계 — 에러 전달 불가, 조기 중단 불가 — 를 정확히 메운다.

```go
func fetchAllStrict(ctx context.Context, urls []string) ([]string, error) {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(10) // 동시 실행 상한. 4번의 세마포어를 대체한다.

	results := make([]string, len(urls))

	for i, url := range urls {
		g.Go(func() error {
			res, err := fetchCtx(ctx, url)
			if err != nil {
				return fmt.Errorf("fetch %s: %w", url, err)
			}
			results[i] = res
			return nil
		})
	}

	if err := g.Wait(); err != nil {
		return nil, err // 첫 번째로 발생한 에러
	}
	return results, nil
}
```

`errgroup.WithContext`가 반환하는 `ctx`를 **반드시 하위 작업에 넘겨야 한다.** 어느 goroutine이든 non-nil 에러를 반환하는 순간 이 `ctx`가 취소되고, 그 신호를 받은 나머지 작업들이 스스로 중단한다. 원래 `ctx`를 그대로 쓰면 조기 중단 기능이 통째로 죽는다.

`g.Wait()`는 **가장 먼저 발생한** 에러 하나만 돌려준다. 전부 모아야 한다면 `errors.Join`으로 직접 수집하거나, 각 결과를 슬롯에 담아두고 나중에 합치면 된다.

## 흔한 함정들

**루프 변수 캡처는 이제 함정이 아니다.** Go 1.21 이하에서는 `for _, v := range s`의 `v`가 루프 전체에서 하나의 변수였고, goroutine이 실행될 즈음엔 이미 마지막 값으로 덮여 있었다. 그래서 `v := v` 같은 재선언 관용구가 필요했다. **Go 1.22부터 루프 변수는 매 반복마다 새로 만들어진다.** 이 글의 예제들이 `v := v` 없이도 올바른 이유다. 다만 `go.mod`의 `go` 지시어가 1.22 미만이면 옛 동작이 유지되니 확인이 필요하다.

**무버퍼 채널의 자기 자신 데드락.** 같은 goroutine에서 보내고 받으면 즉시 데드락이다.

```go
ch := make(chan int) // 버퍼 없음
ch <- 1              // fatal error: all goroutines are asleep - deadlock!
<-ch
```

무버퍼 채널의 송신은 **수신자가 준비될 때까지** 블록된다. 받는 코드가 다음 줄에 있어도 그 줄까지 도달할 수 없다.

**`nil` 채널은 영원히 블록된다.** 버그처럼 보이지만 의도적으로 쓸 수 있다. `select` 안에서 특정 case를 동적으로 끄고 싶을 때 해당 채널 변수를 `nil`로 만들면, 그 case는 영원히 선택되지 않는다.

**goroutine 누수는 조용하다.** 컴파일러도 런타임도 경고하지 않는다. 프로세스가 끝날 때까지 메모리를 붙들고 있을 뿐이다. 테스트에서 잡으려면 [`go.uber.org/goleak`](https://github.com/uber-go/goleak)을 `TestMain`에 걸어두면 된다.

```go
func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m)
}
```

## 정리

패턴 선택은 결국 두 가지 질문으로 좁혀진다.

| 필요한 것 | 쓸 것 |
|---|---|
| 전부 끝날 때까지 대기만 | `sync.WaitGroup` |
| 에러 수집 + 첫 실패 시 전체 취소 | `errgroup` |
| 동시 실행 개수 제한 | `errgroup.SetLimit` 또는 버퍼 채널 세마포어 |
| 타임아웃 / 외부 취소 | `context` + `select` |
| 스트리밍 파이프라인 | 방향성 채널 + `select`로 감싼 송신 |

그리고 어떤 패턴을 쓰든 마지막에 한 번 자문해볼 것: **내가 띄운 goroutine은 어떤 경로로든 반드시 끝나는가?** 답이 "정상 경로에서는"이라면, 아직 끝난 게 아니다.
