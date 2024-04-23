fun main() {
    val ticketNumber = "123321"
    val firstHalf = ticketNumber.substring(0, 3).toList().map { it.toString().toInt() }
    val secondHalf = ticketNumber.substring(3).toList().map { it.toString().toInt() }

    val firstHalfSum = firstHalf.sum()
    val secondHalfSum = secondHalf.sum()

    if (firstHalfSum == secondHalfSum) {
        println("Это счастливый билет!")
    } else {
        println("Это обычный билет :(")
    }
}
