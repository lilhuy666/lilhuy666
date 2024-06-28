Вот простой пример реализации паттерна "Observer" на Java:

import java.util.Observable;
import java.util.Observer;

// Наблюдаемый объект
class ObservableObject extends Observable {
    private String message;

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
        setChanged();
        notifyObservers(message);
    }
}

// Наблюдатель
class ObserverExample implements Observer {
    @Override
    public void update(Observable o, Object arg) {
        System.out.println("Received update: " + arg);
    }
}

public class Main {
    public static void main(String[] args) {
        ObservableObject observable = new ObservableObject();
        ObserverExample observer = new ObserverExample();

        observable.addObserver(observer);
        observable.setMessage("Hello, World!");
    }
}
