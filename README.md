fun main() {
    val ticketNumber = "123456"

    if (isTicketHappy(ticketNumber)) {
        println("Это счастливый билет!")
    } else {
        println("Это обычный билет :(")
    }
}

fun isTicketHappy(ticketNumber: String): Boolean {
    if (ticketNumber.length != 6) {
        return false
    }

    val firstHalf = ticketNumber.substring(0, 3)
    val secondHalf = ticketNumber.substring(3)

    val sumFirstHalf = firstHalf.toCharArray().map { it.toString().toInt() }.sum()
    val sumSecondHalf = secondHalf.toCharArray().map { it.toString().toInt() }.sum()

    return sumFirstHalf == sumSecondHalf
}
