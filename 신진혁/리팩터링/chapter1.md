## 리팩터링: 첫 번째 예시

```kotlin
import java.text.NumberFormat
import java.util.Locale

// Invoice.kt
data class Invoice(
    val customer: String,
    val performances: List<Performance>
)

// Performance.kt
data class Performance(
    val playID: String,
    val audience: Int
)

// Play.kt
data class Play(
    val name: String,
    val type: String
)

fun statement(invoice: Invoice, plays: Map<String, Play>): String {
    var totalAmount = 0
    var volumeCredits = 0
    val result = StringBuilder()

    // 결과 문자열 시작
    result.append("청구 내역 (고객명: ${invoice.customer})\\n")

    // 통화 형식 설정
    val formatter = NumberFormat.getCurrencyInstance(Locale.US)

    for (perf in invoice.performances) {
        val play = plays[perf.playID]
            ?: throw IllegalArgumentException("알 수 없는 playID: ${perf.playID}")

        var thisAmount = 0

        when (play.type) {
            "tragedy" -> {
                thisAmount = 40000
                if (perf.audience > 30) {
                    thisAmount += 1000 * (perf.audience - 30)
                }
            }
            "comedy" -> {
                thisAmount = 30000
                if (perf.audience > 20) {
                    thisAmount += 10000 + 500 * (perf.audience - 20)
                }
                thisAmount += 300 * perf.audience
            }
            else -> throw IllegalArgumentException("알 수 없는 장르: ${play.type}")
        }

        // 포인트 적립
        volumeCredits += maxOf(perf.audience - 30, 0)
        if (play.type == "comedy") {
            volumeCredits += perf.audience / 5
        }

        // 청구 내역 추가
        result.append("  ${play.name}: ${formatter.format(thisAmount / 100.0)} (${perf.audience}석)\\n")

        totalAmount += thisAmount
    }

    // 총액과 포인트 추가
    result.append("총액: ${formatter.format(totalAmount / 100.0)}\\n")
    result.append("적립 포인트: ${volumeCredits}점\\n")

    return result.toString()
}

```

### 리팩터링의 첫 단계

- 리팩터링할 코드 영역을 꼼꼼하게 검사해줄 테스트 코드들부터 마련해야 한다.
    
    ```kotlin
    import org.junit.jupiter.api.Assertions.*
    import org.junit.jupiter.api.Test
    import org.junit.jupiter.api.assertThrows
    
    class StatementTest {
    
        @Test
        fun testStatement() {
            // 플레이 데이터 설정
            val plays = mapOf(
                "hamlet" to Play(name = "Hamlet", type = "tragedy"),
                "as-like" to Play(name = "As You Like It", type = "comedy"),
                "othello" to Play(name = "Othello", type = "tragedy")
            )
    
            // 인보이스 데이터 설정
            val invoice = Invoice(
                customer = "BigCo",
                performances = listOf(
                    Performance(playID = "hamlet", audience = 55),
                    Performance(playID = "as-like", audience = 35),
                    Performance(playID = "othello", audience = 40)
                )
            )
    
            // 기대되는 결과
            val expectedStatement = """
                청구 내역 (고객명: BigCo)
                  Hamlet: $400.00 (55석)
                  As You Like It: $580.00 (35석)
                  Othello: $400.00 (40석)
                총액: $1,380.00
                적립 포인트: 47점
            """.trimIndent()
    
            // 실제 함수 호출
            val actualStatement = statement(invoice, plays)
    
            // 결과 검증
            assertEquals(expectedStatement, actualStatement)
        }
    
        @Test
        fun testUnknownPlayType() {
            // 플레이 데이터 설정 (알 수 없는 장르 추가)
            val plays = mapOf(
                "unknown-play" to Play(name = "Unknown Play", type = "unknown")
            )
    
            // 인보이스 데이터 설정
            val invoice = Invoice(
                customer = "TestCo",
                performances = listOf(
                    Performance(playID = "unknown-play", audience = 10)
                )
            )
    
            // 함수 호출 - 예외 발생 예상
            val exception = assertThrows<IllegalArgumentException> {
                statement(invoice, plays)
            }
    
            // 예외 메시지 검증 (선택 사항)
            assertEquals("알 수 없는 장르: unknown", exception.message)
        }
    
        @Test
        fun testZeroAudience() {
            // 플레이 데이터 설정
            val plays = mapOf(
                "hamlet" to Play(name = "Hamlet", type = "tragedy")
            )
    
            // 인보이스 데이터 설정 (관객 0명)
            val invoice = Invoice(
                customer = "EmptyAudienceCo",
                performances = listOf(
                    Performance(playID = "hamlet", audience = 0)
                )
            )
    
            // 기대되는 결과
            val expectedStatement = """
                청구 내역 (고객명: EmptyAudienceCo)
                  Hamlet: $400.00 (0석)
                총액: $400.00
                적립 포인트: 0점
            """.trimIndent()
    
            // 실제 함수 호출
            val actualStatement = statement(invoice, plays)
    
            // 결과 검증
            assertEquals(expectedStatement, actualStatement)
        }
    }
    
    ```
    

### 함수 쪼개기

1. **‘전체 동작’을 ‘각각의 부분’으로 나눌 수 있는 지점을 찾는다.**
    
    ```kotlin
    when (play.type) {
        "tragedy" -> {
            thisAmount = 40000
            if (perf.audience > 30) {
                thisAmount += 1000 * (perf.audience - 30)
            }
        }
        "comedy" -> {
            thisAmount = 30000
            if (perf.audience > 20) {
                thisAmount += 10000 + 500 * (perf.audience - 20)
            }
            thisAmount += 300 * perf.audience
        }
        else -> throw IllegalArgumentException("알 수 없는 장르: ${play.type}")
    }
    ```
    
2. **함수 추출하기, 코드가 하는 일을 설명하는 이름을 지어준다.**
    - 함수로 빼냈을 때 새 함수에서는 곧바로 사용할 수 없는 유효범위를 벗어나는 변수를 매개 변수로 빼낸다.
        
        ```kotlin
        private fun amountFor(
            play: Play,
            thisAmount: Int,
            perf: Performance
        ): Int {
            ...
        }
        ```
        
    - `perf`와 `play`는 값을 변경하지 않기 때문에 매개변수로 전달하면 된다. `thisAmount`는 함수 안에서 값이 바뀐다. 이 값을 반환한다.
        
        ```kotlin
        private fun amountFor(
            play: Play,
            perf: Performance
        ): Int {
            var thisAmount = 0
        		when (play.type) {
        		    "tragedy" -> {
        		        thisAmount = 40000
        		        if (perf.audience > 30) {
        		            thisAmount += 1000 * (perf.audience - 30)
        		        }
        		    }
        		    "comedy" -> {
        		        thisAmount = 30000
        		        if (perf.audience > 20) {
        		            thisAmount += 10000 + 500 * (perf.audience - 20)
        		        }
        		        thisAmount += 300 * perf.audience
        		    }
        		    else -> throw IllegalArgumentException("알 수 없는 장르: ${play.type}")
        		}
            return thisAmount
        }
        ```
        
    - 추출한 함수를 중첩 함수로 사용한다.
        
        ```kotlin
        import java.text.NumberFormat
        import java.util.Locale
        
        // Invoice.kt
        data class Invoice(
            val customer: String,
            val performances: List<Performance>
        )
        
        // Performance.kt
        data class Performance(
            val playID: String,
            val audience: Int
        )
        
        // Play.kt
        data class Play(
            val name: String,
            val type: String
        )
        
        fun statement(invoice: Invoice, plays: Map<String, Play>): String {
            fun amountFor(
                performance: Performance,
                play: Play,
            ): Int {
                var result = 0
                when (play.type) {
                    "tragedy" -> {
                        result = 40000
                        if (performance.audience > 30) {
                            result += 1000 * (performance.audience - 30)
                        }
                    }
        
                    "comedy" -> {
                        result = 30000
                        if (performance.audience > 20) {
                            result += 10000 + 500 * (performance.audience - 20)
                        }
                        result += 300 * performance.audience
                    }
        
                    else -> throw IllegalArgumentException("알 수 없는 장르: ${play.type}")
                }
                return result
            }
        
            var totalAmount = 0
            var volumeCredits = 0
            val result = StringBuilder()
        
            // 결과 문자열 시작
            result.append("청구 내역 (고객명: ${invoice.customer})\n")
        
            // 통화 형식 설정
            val formatter = NumberFormat.getCurrencyInstance(Locale.US)
        
            for (perf in invoice.performances) {
                val play = plays[perf.playID]
                    ?: throw IllegalArgumentException("알 수 없는 playID: ${perf.playID}")
        
                var thisAmount = 0
        
                thisAmount = amountFor(
                    performance = perf,
                    play = play,
                )
        
                // 포인트 적립
                volumeCredits += maxOf(perf.audience - 30, 0)
                if (play.type == "comedy") {
                    volumeCredits += perf.audience / 5
                }
        
                // 청구 내역 추가
                result.append("  ${play.name}: ${formatter.format(thisAmount / 100.0)} (${perf.audience}석)\n")
        
                totalAmount += thisAmount
            }
        
            // 총액과 포인트 추가
            result.append("총액: ${formatter.format(totalAmount / 100.0)}\n")
            result.append("적립 포인트: ${volumeCredits}점\n")
        
            return result.toString()
        }
        ```
        
    - 추출한 함수의 변수명을 더 명확하게 수정한다.
        
        ```kotlin
        fun amountFor(
            **performance: Perfor,**
            play: Play,
        ): Int {
            **var result = 0**
        		when (play.type) {
        		    "tragedy" -> {
        		        **result** = 40000
        		        if (**performance**.audience > 30) {
        		            thisAmount += 1000 * (**performance**.audience - 30)
        		        }
        		    }
        		    "comedy" -> {
        		        **result** = 30000
        		        if (**performance**.audience > 20) {
        		            thisAmount += 10000 + 500 * (**performance**.audience - 20)
        		        }
        		        **result** += 300 * **performance**.audience
        		    }
        		    else -> throw IllegalArgumentException("알 수 없는 장르: ${play.type}")
        		}
            return **result**
        }
        ```
        
    - play는 performance에 종속적인 변수다 → 매개변수로 전달할 필요가 없다. → 제거한다.
        
        ```kotlin
        fun playFor(
            performance: Performance
        ): Play {
            return plays[performance.playID]
                ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
        }
        ...
        val play = **playFor(perf)**
        var thisAmount = amountFor(
            performance = perf,
            play = play,
        )
        ...
        ```
        
    - 변수 인라인하기
        
        ```kotlin
        for (perf in invoice.performances) {
            var thisAmount = amountFor(
                performance = perf,
                play = **playFor(perf),**
            )
        
            // 포인트 적립
            volumeCredits += maxOf(perf.audience - 30, 0)
            if (**playFor(perf)**.type == "comedy") {
                volumeCredits += perf.audience / 5
            }
        
            // 청구 내역 추가
            result.append("  ${**playFor(perf)**.name}: ${formatter.format(thisAmount / 100.0)} (${perf.audience}석)\n")
        
            totalAmount += thisAmount
        }
        ```
        
    - 변수 인라인했기 때문에 amountFor의 함수 선언을 변경한다.
        
        ```kotlin
        fun amountFor(
            performance: Performance,
        ): Int {
            var result = 0
            when (playFor(performance).type) {
                "tragedy" -> {
                    result = 40000
                    if (performance.audience > 30) {
                        result += 1000 * (performance.audience - 30)
                    }
                }
        
                "comedy" -> {
                    result = 30000
                    if (performance.audience > 20) {
                        result += 10000 + 500 * (performance.audience - 20)
                    }
                    result += 300 * performance.audience
                }
        
                else -> throw IllegalArgumentException("알 수 없는 장르: ${playFor(performance).type}")
            }
            return result
        }
        ...
        for (perf in invoice.performances) {
            // 포인트 적립
            volumeCredits += maxOf(perf.audience - 30, 0)
            if (playFor(perf).type == "comedy") {
                volumeCredits += perf.audience / 5
            }
        
            // 청구 내역 추가
            result.append(
                "  ${playFor(perf).name}: ${formatter.format(**amountFor(perf)** / 100.0)} (${perf.audience}석)\n"
            )
        
            totalAmount += **amountFor(perf)**
        }
        ...
        ```
        
3. **1번과 2번 과정을 반복한다.**
    
    ```kotlin
    var volumeCredits = 0
    ...
    volumeCredits += maxOf(perf.audience - 30, 0)
    if (play.type == "comedy") {
        volumeCredits += perf.audience / 5
    }
    ```
    
    - 함수명, 변수명 변경
        
        ```kotlin
        fun statement(invoice: Invoice, plays: Map<String, Play>): String {
            **fun playFor(**
                performance: Performance
            ): Play {
                return plays[performance.playID]
                    ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
            **}**
        
            **fun amountFor(**
                performance: Performance,
            ): Int {
                var result = 0
                when (playFor(performance).type) {
                    "tragedy" -> {
                        result = 40000
                        if (performance.audience > 30) {
                            result += 1000 * (performance.audience - 30)
                        }
                    }
        
                    "comedy" -> {
                        result = 30000
                        if (performance.audience > 20) {
                            result += 10000 + 500 * (performance.audience - 20)
                        }
                        result += 300 * performance.audience
                    }
        
                    else -> throw IllegalArgumentException("알 수 없는 장르: ${playFor(performance).type}")
                }
                return result
            **}**
        
            **fun volumeCreditsFor(performance: Performance): Int {**
                var result = 0
                // 포인트 적립
                result += maxOf(performance.audience - 30, 0)
                if (playFor(performance).type == "comedy") {
                    result += performance.audience / 5
                }
                return result
            **}**
        
            var totalAmount = 0
            var volumeCredits = 0
            val result = StringBuilder()
        
            // 결과 문자열 시작
            result.append("청구 내역 (고객명: ${invoice.customer})\n")
        
            // 통화 형식 설정
            val formatter = NumberFormat.getCurrencyInstance(Locale.US)
        
            for (perf in invoice.performances) {
                // 포인트 적립
                volumeCredits += volumeCreditsFor(perf)
                // 청구 내역 추가
                result.append(
                    "  ${playFor(perf).name}: ${formatter.format(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
                )
        
                totalAmount += amountFor(perf)
            }
        
            // 총액과 포인트 추가
            result.append("총액: ${formatter.format(totalAmount / 100.0)}\n")
            result.append("적립 포인트: ${volumeCredits}점\n")
        
            return result.toString()
        }
        ```
        
4. **임시 변수 제거 (나중에 문제가 될 수 있다.)**
    
    ```kotlin
    // 통화 형식 설정
    val formatter = NumberFormat.getCurrencyInstance(Locale.US)
    
    for (perf in invoice.performances) {
        // 포인트 적립
        volumeCredits += volumeCreditsFor(perf)
        // 청구 내역 추가
        result.append(
            "  ${playFor(perf).name}: ${formatter.format(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
        )
    
        totalAmount += amountFor(perf)
    }
    
    // 총액과 포인트 추가
    result.append("총액: ${formatter.format(totalAmount / 100.0)}\n")
    ```
    
    - 함수명, 변수명 변경 (이름을 잘지어야만 효과가 있다.)
        
        ```kotlin
        fun statement(invoice: Invoice, plays: Map<String, Play>): String {
        		...
            **fun usd(number: Double): String =**
                NumberFormat
                    .getCurrencyInstance(Locale.US)
                    .format(number)
        
            var totalAmount = 0
            var volumeCredits = 0
            val result = StringBuilder()
        
            // 결과 문자열 시작
            result.append("청구 내역 (고객명: ${invoice.customer})\n")
            for (perf in invoice.performances) {
                // 포인트 적립
                volumeCredits += volumeCreditsFor(perf)
                // 청구 내역 추가
                result.append(
                    "  ${playFor(perf).name}: ${**usd(amountFor(perf) / 100.0)**} (${perf.audience}석)\n"
                )
        
                totalAmount += amountFor(perf)
            }
        
            // 총액과 포인트 추가
            result.append("총액: ${**usd(totalAmount / 100.0)**}\n")
            result.append("적립 포인트: ${volumeCredits}점\n")
        
            return result.toString()
        }
        ```
        

### 반복문 쪼개기

1. **값 누적 로직을 분리**
    
    ```kotlin
    for (perf in invoice.performances) {
        // 포인트 적립
        volumeCredits += volumeCreditsFor(perf)
        // 청구 내역 추가
        result.append(
            "  ${playFor(perf).name}: ${usd(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
        )
    
        totalAmount += amountFor(perf)
    }
    
    **for (perf in invoice.performances) {
        // 포인트 적립
        volumeCredits += volumeCreditsFor(perf)
    }**
    ```
    
    - 문장 슬라이드하기
        
        ```kotlin
        var totalAmount = 0
        val result = StringBuilder()
        
        // 결과 문자열 시작
        result.append("청구 내역 (고객명: ${invoice.customer})\n")
        for (perf in invoice.performances) {
            // 청구 내역 추가
            result.append(
                "  ${playFor(perf).name}: ${usd(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
            )
        
            totalAmount += amountFor(perf)
        }
        
        **var volumeCredits = 0
        for (perf in invoice.performances) {
            // 포인트 적립
            volumeCredits += volumeCreditsFor(perf)
        }**
        
        // 총액과 포인트 추가
        result.append("총액: ${usd(totalAmount / 100.0)}\n")
        result.append("적립 포인트: ${volumeCredits}점\n")
        
        return result.toString()
        ```
        
    - 함수 추출하기, 함수명, 변수명 수정하기, 인라인하기
        
        ```kotlin
        fun statement(invoice: Invoice, plays: Map<String, Play>): String {
            fun playFor(
                performance: Performance
            ): Play {
                return plays[performance.playID]
                    ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
            }
        
            fun amountFor(
                performance: Performance,
            ): Int {
                var result = 0
                when (playFor(performance).type) {
                    "tragedy" -> {
                        result = 40000
                        if (performance.audience > 30) {
                            result += 1000 * (performance.audience - 30)
                        }
                    }
        
                    "comedy" -> {
                        result = 30000
                        if (performance.audience > 20) {
                            result += 10000 + 500 * (performance.audience - 20)
                        }
                        result += 300 * performance.audience
                    }
        
                    else -> throw IllegalArgumentException("알 수 없는 장르: ${playFor(performance).type}")
                }
                return result
            }
        
            fun volumeCreditsFor(performance: Performance): Int {
                var result = 0
                // 포인트 적립
                result += maxOf(performance.audience - 30, 0)
                if (playFor(performance).type == "comedy") {
                    result += performance.audience / 5
                }
                return result
            }
        
            fun usd(number: Double): String =
                NumberFormat
                    .getCurrencyInstance(Locale.US)
                    .format(number)
        
            **fun totalVolumeCredits(): Int {
                var result = 0
                for (perf in invoice.performances) {
                    // 포인트 적립
                    result += volumeCreditsFor(perf)
                }
                return result
            }**
        
            var totalAmount = 0
            val result = StringBuilder()
        
            // 결과 문자열 시작
            result.append("청구 내역 (고객명: ${invoice.customer})\n")
            for (perf in invoice.performances) {
                // 청구 내역 추가
                result.append(
                    "  ${playFor(perf).name}: ${usd(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
                )
        
                totalAmount += amountFor(perf)
            }
        
            // 총액과 포인트 추가
            result.append("총액: ${usd(totalAmount / 100.0)}\n")
            result.append("적립 포인트: ${**totalVolumeCredits()**}점\n")
        
            return result.toString()
        }
        ```
        
        - **이 정도 중복은 성능에 큰 영향을 미치지 않는다.**
            
            → **특별한 경우가 아니라면 일단 무시하라**
            
2. **1번 과정을 반복한다.**
    
    ```kotlin
    for (perf in invoice.performances) {
        // 청구 내역 추가
        result.append(
            "  ${playFor(perf).name}: ${usd(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
        )
    }
    for (perf in invoice.performances) {
        totalAmount += amountFor(perf)
    }
    ```
    
    - 문장 슬라이드, 함수 추출하기, 함수명, 변수명 수정하기, 인라인하기

```kotlin
**fun statement(invoice: Invoice, plays: Map<String, Play>): String {**
    fun playFor(
        performance: Performance
    ): Play {
        return plays[performance.playID]
            ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
    }

    fun amountFor(
        performance: Performance,
    ): Int {
        var result = 0
        when (playFor(performance).type) {
            "tragedy" -> {
                result = 40000
                if (performance.audience > 30) {
                    result += 1000 * (performance.audience - 30)
                }
            }

            "comedy" -> {
                result = 30000
                if (performance.audience > 20) {
                    result += 10000 + 500 * (performance.audience - 20)
                }
                result += 300 * performance.audience
            }

            else -> throw IllegalArgumentException("알 수 없는 장르: ${playFor(performance).type}")
        }
        return result
    }

    fun volumeCreditsFor(performance: Performance): Int {
        var result = 0
        // 포인트 적립
        result += maxOf(performance.audience - 30, 0)
        if (playFor(performance).type == "comedy") {
            result += performance.audience / 5
        }
        return result
    }

    fun usd(number: Double): String =
        NumberFormat
            .getCurrencyInstance(Locale.US)
            .format(number)

    fun totalVolumeCredits(): Int {
        var result = 0
        for (perf in invoice.performances) {
            // 포인트 적립
            result += volumeCreditsFor(perf)
        }
        return result
    }

    fun totalAmount(): Int {
        var result = 0
        for (perf in invoice.performances) {
            result += amountFor(perf)
        }
        return result
    }

    **val result = StringBuilder()
    // 결과 문자열 시작
    result.append("청구 내역 (고객명: ${invoice.customer})\n")
    for (perf in invoice.performances) {
        // 청구 내역 추가
        result.append(
            "  ${playFor(perf).name}: ${usd(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
        )
    }

    // 총액과 포인트 추가
    result.append("총액: ${usd(totalAmount() / 100.0)}\n")
    result.append("적립 포인트: ${totalVolumeCredits()}점\n")

    return result.toString()**
**}**
```

### 중간 점검: 난무하는 중첩 함수 (다음 단계)
