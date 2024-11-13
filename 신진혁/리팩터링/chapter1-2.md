### 중간 점검: 난무하는 중첩 함수

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

### 계산 단계와 포맷팅 단계 분리하기

구조 개선이 끝나고 **단계 쪼개기**를 한다.

첫 번째 단계 : statement()에 필요한 데이터를 처리한다. (두 번째 단계에서 필요한 데이터 구조를 생성)

두 번째 단계 : 첫 번째 단계의 결과를 텍스트나 HTML로 표현한다.

1. 두 번째 단계가 될 코드들을 함수 추출하기로 뽑아낸다.
    - 청구 내역을 출력하는 코드(HTML로 표현하는 코드) statement()(본문 전체)를 추출하자
        
        ```kotlin
        class Statement {
            
            **fun statement(invoice: Invoice, plays: Map<String, Play>): String {
                return renderPlainText(plays, invoice)
            }**
        
            private fun renderPlainText(
                plays: Map<String, Play>,
                invoice: Invoice
            ): String {
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
        
                val result = StringBuilder()
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
        
                return result.toString()
            }
        }
        
        ```
        
2. 기존 인자로부터 중간 데이터 구조 역할을 할 객체를 생성한 후 새로운 인자로 전달
    
    ```kotlin
    **data class StatementData(
        val customer: String,
        val performances: List<Performance>
    )**
    
    class Statement {
    
        fun statement(invoice: Invoice, plays: Map<String, Play>): String {
            **val statementData = StatementData(
                customer = invoice.customer,
                performances = invoice.performances
            )**
            return renderPlainText(plays, **statementData**)
        }
    
        private fun renderPlainText(
            plays: Map<String, Play>,
            **data: StatementData**
        ): String {
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
                for (perf in data.performances) {
                    // 포인트 적립
                    result += volumeCreditsFor(perf)
                }
                return result
            }
    
            fun totalAmount(): Int {
                var result = 0
                for (perf in data.performances) {
                    result += amountFor(perf)
                }
                return result
            }
    
            val result = StringBuilder()
            // 결과 문자열 시작
            result.append("청구 내역 (고객명: ${data.customer})\n")
            for (perf in data.performances) {
                // 청구 내역 추가
                result.append(
                    "  ${playFor(perf).name}: ${usd(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
                )
            }
    
            // 총액과 포인트 추가
            result.append("총액: ${usd(totalAmount() / 100.0)}\n")
            result.append("적립 포인트: ${totalVolumeCredits()}점\n")
    
            return result.toString()
        }
    }
    
    ```
    
    - List와 같은 가변 데이터는 얕은 복사를 통해 불변처럼 취급한다.
        
        ```kotlin
        fun statement(invoice: Invoice, plays: Map<String, Play>): String {
        		**fun enrichPerformance(performance: Performance): Performance {
        		    return performance.copy()
        		}**
        
            val statementData = StatementData(
                customer = invoice.customer,
                **performances = invoice.performances.map(::enrichPerformance)**
            )
            return renderPlainText(plays, statementData)
        }
        ```
        
3. 함수를 옮긴다.
    - **playFor** 메서드를 enrichPerformance 내부 호출이 가능하게 옮긴다.
        
        ```kotlin
        // Play.kt
        data class Play(
            val name: String,
            val type: String
        )
        
        data class StatementData(
            val customer: String,
            val performances: List<Performance>
        )
        
        class Statement {
        
            fun statement(invoice: Invoice, plays: Map<String, Play>): String {
                **fun playFor(
                    performance: Performance
                ): Play {
                    return plays[performance.playID]
                        ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
                }**
        
                fun enrichPerformance(performance: Performance): Performance {
                    val result = performance.copy(
                        play = **playFor**(performance)
                    )
                    return result
                }
        
                val statementData = StatementData(
                    customer = invoice.customer,
                    performances = invoice.performances.map(::**enrichPerformance**)
                )
                return renderPlainText(plays, statementData)
            }
            
            private fun renderPlainText(
                plays: Map<String, Play>,
                data: StatementData
            ): String {
                fun amountFor(
                    performance: Performance,
                ): Int {
                    var result = 0
                    when (**performance.play.type**) {
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
        
                        else -> throw IllegalArgumentException("알 수 없는 장르: ${**performance.play**.type}")
                    }
                    return result
                }
        
                fun volumeCreditsFor(performance: Performance): Int {
                    var result = 0
                    // 포인트 적립
                    result += maxOf(performance.audience - 30, 0)
                    if (performance.play.type == "comedy") {
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
                    for (perf in data.performances) {
                        // 포인트 적립
                        result += volumeCreditsFor(perf)
                    }
                    return result
                }
        
                fun totalAmount(): Int {
                    var result = 0
                    for (perf in data.performances) {
                        result += amountFor(perf)
                    }
                    return result
                }
        
                val result = StringBuilder()
                // 결과 문자열 시작
                result.append("청구 내역 (고객명: ${data.customer})\n")
                for (perf in data.performances) {
                    // 청구 내역 추가
                    result.append(
                        "  ${**perf.play.name**}: ${usd(amountFor(perf) / 100.0)} (${perf.audience}석)\n"
                    )
                }
        
                // 총액과 포인트 추가
                result.append("총액: ${usd(totalAmount() / 100.0)}\n")
                result.append("적립 포인트: ${totalVolumeCredits()}점\n")
        
                return result.toString()
            }
        }
        ```
        
    - amounFor() 메서드도 옮긴다.
        
        ```kotlin
        // Performance.kt
        data class Performance(
            **val play: Play,**
            val playID: String,
            val audience: Int,
            **val amount: Int**
        )
        
        // Play.kt
        data class Play(
            val name: String,
            val type: String
        )
        
        data class StatementData(
            val customer: String,
            val performances: List<Performance>
        )
        
        class Statement {
        
            fun statement(invoice: Invoice, plays: Map<String, Play>): String {
                fun amountFor(
                    performance: Performance,
                ): Int {
                    var result = 0
                    when (performance.play.type) {
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
        
                        else -> throw IllegalArgumentException("알 수 없는 장르: ${performance.play.type}")
                    }
                    return result
                }
        
                fun playFor(
                    performance: Performance
                ): Play {
                    return plays[performance.playID]
                        ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
                }
        
                fun enrichPerformance(performance: Performance): Performance {
                    val result = performance.copy(
                        play = playFor(performance),
                        amount = amountFor(performance)
                    )
                    return result
                }
        
                val statementData = StatementData(
                    customer = invoice.customer,
                    performances = invoice.performances.map(::enrichPerformance)
                )
                return renderPlainText(plays, statementData)
            }
        
            private fun renderPlainText(
                plays: Map<String, Play>,
                data: StatementData
            ): String {
                fun volumeCreditsFor(performance: Performance): Int {
                    var result = 0
                    // 포인트 적립
                    result += maxOf(performance.audience - 30, 0)
                    if (performance.play.type == "comedy") {
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
                    for (perf in data.performances) {
                        // 포인트 적립
                        result += volumeCreditsFor(perf)
                    }
                    return result
                }
        
                fun totalAmount(): Int {
                    var result = 0
                    for (perf in data.performances) {
                        result += **perf.amount**
                    }
                    return result
                }
        
                val result = StringBuilder()
                // 결과 문자열 시작
                result.append("청구 내역 (고객명: ${data.customer})\n")
                for (perf in data.performances) {
                    // 청구 내역 추가
                    result.append(
                        "  ${perf.play.name}: ${usd(**perf.amount**.toDouble())} (${perf.audience}석)\n"
                    )
                }
        
                // 총액과 포인트 추가
                result.append("총액: ${usd(totalAmount() / 100.0)}\n")
                result.append("적립 포인트: ${totalVolumeCredits()}점\n")
        
                return result.toString()
            }
        }
        ```
        
    - volumeCreditsFor 메서드도 옮긴다.
        
        ```kotlin
        // Performance.kt
        data class Performance(
        val play: Play,
        val playID: String,
        val audience: Int,
        val amount: Int
        val volumeCredits: Int
        )
        
        // Play.kt
        data class Play(
        val name: String,
        val type: String
        )
        
        data class StatementData(
        val customer: String,
        val performances: List<Performance>
        )
        
        class Statement {
        
        fun statement(invoice: Invoice, plays: Map<String, Play>): String {
            fun volumeCreditsFor(performance: Performance): Int {
                var result = 0
                // 포인트 적립
                result += maxOf(performance.audience - 30, 0)
                if (performance.play.type == "comedy") {
                    result += performance.audience / 5
                }
                return result
            }
            
            fun amountFor(
                performance: Performance,
            ): Int {
                var result = 0
                when (performance.play.type) {
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
        
                    else -> throw IllegalArgumentException("알 수 없는 장르: ${performance.play.type}")
                }
                return result
            }
        
            fun playFor(
                performance: Performance
            ): Play {
                return plays[performance.playID]
                    ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
            }
        
            fun enrichPerformance(performance: Performance): Performance {
                val result = performance.copy(
                    play = playFor(performance),
                    amount = amountFor(performance),
                    volumeCredits = volumeCreditsFor(performance)
                )
                return result
            }
        
            val statementData = StatementData(
                customer = invoice.customer,
                performances = invoice.performances.map(::enrichPerformance)
            )
            return renderPlainText(plays, statementData)
        }
        
        private fun renderPlainText(
            plays: Map<String, Play>,
            data: StatementData
        ): String {
            fun usd(number: Double): String =
                NumberFormat
                    .getCurrencyInstance(Locale.US)
                    .format(number)
        
            fun totalVolumeCredits(): Int {
                var result = 0
                for (perf in data.performances) {
                    // 포인트 적립
                    **result += perf.volumeCredits**
                }
                return result
            }
        
            fun totalAmount(): Int {
                var result = 0
                for (perf in data.performances) {
                    result += perf.amount
                }
                return result
            }
        
            val result = StringBuilder()
            // 결과 문자열 시작
            result.append("청구 내역 (고객명: ${data.customer})\n")
            for (perf in data.performances) {
                // 청구 내역 추가
                result.append(
                    "  ${perf.play.name}: ${usd(perf.amount.toDouble())} (${perf.audience}석)\n"
                )
            }
        
            // 총액과 포인트 추가
            result.append("총액: ${usd(totalAmount() / 100.0)}\n")
            result.append("적립 포인트: ${totalVolumeCredits()}점\n")
        
            return result.toString()
        }
        ```
        
    - 계산 총합 구하는 메서드 옮기기
        
        ```kotlin
        data class Performance(
            val play: Play,
            val playID: String,
            val audience: Int,
            val amount: Int,
            val volumeCredits: Int,
        )
        
        // Play.kt
        data class Play(
            val name: String,
            val type: String
        )
        
        **data class StatementData(
            var customer: String? = null,
            var performances: List<Performance> = emptyList(),
            var totalVolumeCredits: Int? = null,
            var totalAmount: Int? = null,
        )**
        
        class Statement {
        
            fun statement(invoice: Invoice, plays: Map<String, Play>): String {
                **fun totalVolumeCredits(data: StatementData): Int {
                    var result = 0
                    for (perf in data.performances) {
                        // 포인트 적립
                        result += perf.volumeCredits
                    }
                    return result
                }
        
                fun totalAmount(data: StatementData): Int {
                    var result = 0
                    for (perf in data.performances) {
                        result += perf.amount
                    }
                    return result
                }**
        
                fun volumeCreditsFor(performance: Performance): Int {
                    var result = 0
                    // 포인트 적립
                    result += maxOf(performance.audience - 30, 0)
                    if (performance.play.type == "comedy") {
                        result += performance.audience / 5
                    }
                    return result
                }
        
                fun amountFor(
                    performance: Performance,
                ): Int {
                    var result = 0
                    when (performance.play.type) {
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
        
                        else -> throw IllegalArgumentException("알 수 없는 장르: ${performance.play.type}")
                    }
                    return result
                }
        
                fun playFor(
                    performance: Performance
                ): Play {
                    return plays[performance.playID]
                        ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
                }
        
                fun enrichPerformance(performance: Performance): Performance {
                    val result = performance.copy(
                        play = playFor(performance),
                        amount = amountFor(performance),
                        volumeCredits = volumeCreditsFor(performance)
                    )
                    return result
                }
        
                val statementData = StatementData()
        
                statementData.customer = invoice.customer
                statementData.performances = invoice.performances.map(::enrichPerformance)
                **statementData.totalAmount = totalAmount(data = statementData)
                statementData.totalVolumeCredits = totalVolumeCredits(data = statementData)**
        
                return renderPlainText(plays, statementData)
            }
        
            private fun renderPlainText(
                plays: Map<String, Play>,
                data: StatementData
            ): String {
                fun usd(number: Double): String =
                    NumberFormat
                        .getCurrencyInstance(Locale.US)
                        .format(number)
        
                val result = StringBuilder()
                // 결과 문자열 시작
                result.append("청구 내역 (고객명: ${data.customer})\n")
                for (perf in data.performances) {
                    // 청구 내역 추가
                    result.append(
                        "  ${perf.play.name}: ${usd(perf.amount.toDouble())} (${perf.audience}석)\n"
                    )
                }
        
                // 총액과 포인트 추가
                **result.append("총액: ${usd((data.totalAmount ?: 0) / 100.0)}\n")
                result.append("적립 포인트: ${data.totalVolumeCredits}점\n")**
        
                return result.toString()
            }
        ```
        
4. 반복문을 파이프 라인으로 바꾸자
    
    ```kotlin
    **fun totalVolumeCredits(data: StatementData): Int {
        return data.performances.sumOf { it.volumeCredits }
    }
    
    fun totalAmount(data: StatementData): Int {
        return data.performances.sumOf { it.amount }
    }**
    ```
    
5. statement()에 필요한 데이터 처리에 해당하는 코드를 모두 별도 함수로 뺌
    
    ```kotlin
    fun statement(invoice: Invoice, plays: Map<String, Play>): String {
        **val statementData = createStatementData(plays, invoice)**
        return renderPlainText(plays, statementData)
    }
        
    **/** 다른 파일 */
    private fun createStatementData(**
        plays: Map<String, Play>,
        invoice: Invoice
    ): StatementData {
        fun totalVolumeCredits(data: StatementData): Int {
            return data.performances.sumOf { it.volumeCredits }
        }
    
        fun totalAmount(data: StatementData): Int {
            return data.performances.sumOf { it.amount }
        }
    
        fun volumeCreditsFor(performance: Performance): Int {
            var result = 0
            // 포인트 적립
            result += maxOf(performance.audience - 30, 0)
            if (performance.play.type == "comedy") {
                result += performance.audience / 5
            }
            return result
        }
    
        fun amountFor(
            performance: Performance,
        ): Int {
            var result = 0
            when (performance.play.type) {
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
    
                else -> throw IllegalArgumentException("알 수 없는 장르: ${performance.play.type}")
            }
            return result
        }
    
        fun playFor(
            performance: Performance
        ): Play {
            return plays[performance.playID]
                ?: throw IllegalArgumentException("알 수 없는 playID: ${performance.playID}")
        }
    
        fun enrichPerformance(performance: Performance): Performance {
            val result = performance.copy(
                play = playFor(performance),
                amount = amountFor(performance),
                volumeCredits = volumeCreditsFor(performance)
            )
            return result
        }
    
        val statementData = StatementData()
        statementData.customer = invoice.customer
        statementData.performances = invoice.performances.map(::enrichPerformance)
        statementData.totalAmount = totalAmount(data = statementData)
        statementData.totalVolumeCredits = totalVolumeCredits(data = statementData)
        return statementData
    }
    ```
