fun main() {
    val ticketNumber = "123321"

    val firstHalfSum = ticketNumber.substring(0, 3).map { it.toString().toInt() }.sum()
    val secondHalfSum = ticketNumber.substring(3).map { it.toString().toInt() }.sum()

    if (firstHalfSum == secondHalfSum) {
        println("Это счастливый билет!")
    } else {
        println("Это обычный билет :(")
    }
}
