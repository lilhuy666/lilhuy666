fun main() {
    val ticketNumber = "123321"
    val ticketDigits = ticketNumber.map { it.toString().toInt() }

    val firstHalfSum = ticketDigits.slice(0..2).sum()
    val secondHalfSum = ticketDigits.slice(3..5).sum()

    if (firstHalfSum == secondHalfSum) {
        println("Это счастливый билет!")
    } else {
        println("Это обычный билет :(")
    }
}
