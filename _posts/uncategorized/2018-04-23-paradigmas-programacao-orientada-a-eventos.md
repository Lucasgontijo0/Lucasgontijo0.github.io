
---
layout: post
title: Programação Orientada a Eventos
categories: [Paradigmas de Programação]
tag: []
---

## O que é ?

Mas antes de entendermos a programação orientada a eventos, precisamos entender o que é um evento: 
Um evento é a indicação de algo que aconteceu, é o resultado de uma ação.

A Programação Orientada à Eventos é um paradigma de programação onde a execução do código é determinada pelo disparo de eventos, existem rotinas responsáveis por capturar estes eventos e executar determinados comportamentos que são respostas a estes eventos. Ou seja, a ocorrência de um evento pode provocar uma reação que pode ser uma ação (ou um conjunto delas) a ser tomada.

Uma curiosidade deste paradigma é que ele trabalha com o Princípio de Hollywood: "**Não nos telefone, nós telefonaremos para você.**"
A princípio seriam atores que realizaram testes e estão aguardando uma ligação com o resultado em vez de ligarem pedindo a informação, por exemplo.

Para contextualizar melhor o conceito realizei uma abordagem baseada em [Enfileiramento](https://github.com/constantlearning/poc-event-driven-programming/tree/master/src/queuing).

## Enfileiramento

Para a melhor visualização de um sistema utilizando a abordagem de enfileiramento de eventos, poderíamos imaginar um cenário de cadastro de clientes, onde há um alto volume de pedidos em uma infraestrutura que não tem poder computacional suficiente para atender à todos os pedidos simultaneamente.

Uma abordagem interessante para solucionar este problema, seria o enfileiramento destes eventos de cadastro para que sejam processados pouco a pouco conforme o poder computacional da infraestrutura for permitindo.

### Implementação

**Motor de Eventos:**
O motor de eventos, possui uma fila para o armazenamento dos eventos que chegam e também para o consumo onde ao invocar o método **start()** é aberta uma thread que fica infinitamente consumindo a fila. Existe também um mapa que utilizei para registrar os tipos de eventos e um tipo genérico (EventListener) que implementa a ação a ser tomada para aquele tipo de evento.
```java
package queuing.event;
import java.util.HashMap;  
import java.util.Map;  
import java.util.Queue;  
import java.util.concurrent.LinkedBlockingQueue;  
  
public class EventEngine {  
  
    private static Queue<Event> eventQueue = new LinkedBlockingQueue<>();  
    private static Map<String, EventListener> eventListener = new HashMap<>();  
  
    public static void subscribeListener(String type, EventListener listener) {  
        eventListener.put(type, listener);  
    }  
  
    public static void publish(Event event) {  
        eventQueue.add(event);  
    }  
  
    public static void start() {  
  
        new Thread(() -> {  
  
            while (true) {  
  
                Event event = eventQueue.poll();  
  
                if (event != null) {  
  
                        EventListener listener = eventListener.get(event.getType());  
  
                    if (listener != null) {  
                        listener.onRecieve(event);  
                    }  
                }  
  
                try {  
                    Thread.sleep(10);  
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
  
                System.out.println("Pool Size: " + eventQueue.size());  
            }  
  
        }).start();  
  }  
}

```
**Evento:**
O evento contem um tipo, que nesta implementação usamos para identificar qual implementação de listener irá processá-lo, contem também uma informação que deverá ser computada, no nosso caso **payload** e também duas abstrações que representam callbacks que é um pedaço de código que é passado como parâmetro e que em algum momento será executado, no nosso caso na falha ou sucesso.
```java
package queuing.event;  
  
public class Event {  
  
    private String type;  
    private Object payload;  
    private Callback onSucess;  
    private Callback onFailure;  
  
    public Event(String type, Object payload, Callback onSucess, Callback onFailure) {  
        this.type = type;  
        this.payload = payload;  
        this.onSucess = onSucess;  
        this.onFailure = onFailure;  
    }  
  
    public String getType() {  
        return type;  
    }  
  
    public Object getPayload() {  
        return payload;  
    }  
  
    public Callback getOnSucess() {  
        return onSucess;  
    }  
  
    public Callback getOnFailure() {  
        return onFailure;  
    }  
}
```

**Callback:**
Recebe um argumento e deve processá-lo, nesta implementação definimos o conteúdo dos callbacks no **RegisterClientListener** e o seu comportamento no main, durante o momento da criação do evento.
```java
package queuing.event;  
  
public interface Callback {  
    void callback(Object payload);  
}
```

**Event Listener:**
Usado para realizar o polimorfismo no algoritmo do motor, assim nós podemos processar qualquer evento, desde que esteja mapeado.
```java
package queuing.event;  
  
public interface EventListener {
    void onRecieve(Event event);  
}

```
**Registro de Clientes Listener:**
Implementa a ação do evento, o que deve ser feito. Se houvessem novos eventos iriamos utilizar a interface **Event Listener** e implementar um comportamento para este novo evento.
```java
package queuing.event;  
  
import queuing.repository.Client;  
import queuing.repository.Repository;  
  
public class RegisterClientListener implements EventListener {  
  
    @Override  
    public void onRecieve(Event event) {  
  
        Repository repository = Repository.getInstance();  
  
        Client client = (Client) event.getPayload();  
        System.out.println("Initializing client registration: " + client);  
  
        Callback callback;  
  
        try {  
            repository.insertClient(client);  
            callback = event.getOnSucess();  
            callback.callback("Success on registration;");  
 
        } catch (RuntimeException e) {  
            callback = event.getOnFailure();  
            callback.callback(e);  
        }  
    }  
}
```

Temos duas classes para representar os clientes a serem cadastrados e uma abstração de repositório onde vamos inserir os clientes que forem chegando.

**Cliente**
```java
package queuing.repository;  
  
import java.util.UUID;  
  
public class Client {  
  
    private Integer id;  
    private UUID name;  
  
    public Client(Integer id, UUID name) {  
        this.id = id;  
        this.name = name;  
    }  
  
    public Integer getId() {  
        return id;  
    }  
  
    @Override  
    public String toString() {  
        return "Client:{" +  
                "id=" + id +  
                ", name=" + name +  
                '}';  
    }  
}
```

**Repositório**
```java
package queuing.repository;  
  
import java.util.ArrayList;  
import java.util.List;  
  
public class Repository {  
  
    private List<Client> clientsRepository;  
  
    private static Repository instance = null;  
  
    public static Repository getInstance() {  
        if (instance == null) {  
            instance = new Repository();  
		}  
        return instance;  
    }  
  
    private Repository() {  
        this.clientsRepository = new ArrayList<>();  
    }  
  
    public void insertClient(Client client) {  
  
        if (client.getId() % 2 != 0) {  
            throw new RuntimeException("Invalid Client!");  
        }  
  
        this.clientsRepository.add(client);  
    }  
}
```

**Main:**
Por fim temos a classe Main, onde iniciamos o motor de evento e cadastramos um tipo de evento. Então realizamos o pedido de cadastro de 1 milhão de clientes, onde cada pedido é um evento que contem um tipo para ser identificado, um payload que é a informação a ser processada neste caso é o cliente, e então a implementação de dois callbacks que serão o que deve ser executado no caso de sucesso ou falha e que vai ser publicado na fila do motor para posteriormente ser consumido.
```java
package queuing;  
  
import queuing.event.Callback;  
import queuing.event.Event;  
import queuing.event.EventEngine;  
import queuing.event.RegisterClientListener;  
import queuing.repository.Client;  
  
import java.util.UUID;  
  
public class Main {  
  
    static {  
        EventEngine.start();  
        EventEngine.subscribeListener("CLIENT_REGISTRATION", new RegisterClientListener());  
      }  
  
    public static void main(String[] args) {  
  
        for (int i = 0; i < 1000000; i++) {  
            Client client = new Client(i, UUID.randomUUID());  
  
            Event event = new Event("CLIENT_REGISTRATION", client,
            new Callback() {  
                @Override  
                public void callback(Object payload) {  
                    String value = (String) payload;  
                    System.out.println("Success callback: " + value);  
                }  
            },  
            new Callback() {  
                @Override  
                public void callback(Object payload) {  
                    RuntimeException exception = (RuntimeException) payload;  
                    System.out.println("Failure callback: " + exception.getMessage());  
                }  
            });  
  
            EventEngine.publish(event);  
        }  
    }  
}
```
> **Observação**: É importante ser observado que por ter um motor de eventos que fica o tempo todo consumindo da fila, ao mesmo tempo em que os eventos estão sendo cadastrados estão também sendo consumidos.

Esse é um exemplo básico, mas dá para ter uma ideia do funcionamento deste paradigma.---
layout: post
title: Programação Orientada a Eventos
categories: [Paradigmas de Programação]
tag: []
---

## O que é ?

Mas antes de entendermos a programação orientada a eventos, precisamos entender o que é um evento: 
Um evento é a indicação de algo que aconteceu, é o resultado de uma ação.

A Programação Orientada à Eventos é um paradigma de programação onde a execução do código é determinada pelo disparo de eventos, existem rotinas responsáveis por capturar estes eventos e executar determinados comportamentos que são respostas a estes eventos. Ou seja, a ocorrência de um evento pode provocar uma reação que pode ser uma ação (ou um conjunto delas) a ser tomada.

Uma curiosidade deste paradigma é que ele trabalha com o Princípio de Hollywood: "**Não nos telefone, nós telefonaremos para você.**"
A princípio seriam atores que realizaram testes e estão aguardando uma ligação com o resultado em vez de ligarem pedindo a informação, por exemplo.

Para contextualizar melhor o conceito realizei uma abordagem baseada em [Enfileiramento](https://github.com/constantlearning/poc-event-driven-programming/tree/master/src/queuing).

## Enfileiramento

Para a melhor visualização de um sistema utilizando a abordagem de enfileiramento de eventos, poderíamos imaginar um cenário de cadastro de clientes, onde há um alto volume de pedidos em uma infraestrutura que não tem poder computacional suficiente para atender à todos os pedidos simultaneamente.

Uma abordagem interessante para solucionar este problema, seria o enfileiramento destes eventos de cadastro para que sejam processados pouco a pouco conforme o poder computacional da infraestrutura for permitindo.

### Implementação

**Motor de Eventos:**
O motor de eventos, possui uma fila para o armazenamento dos eventos que chegam e também para o consumo onde ao invocar o método **start()** é aberta uma thread que fica infinitamente consumindo a fila. Existe também um mapa que utilizei para registrar os tipos de eventos e um tipo genérico (EventListener) que implementa a ação a ser tomada para aquele tipo de evento.
```java
package queuing.event;
import java.util.HashMap;  
import java.util.Map;  
import java.util.Queue;  
import java.util.concurrent.LinkedBlockingQueue;  
  
public class EventEngine {  
  
    private static Queue<Event> eventQueue = new LinkedBlockingQueue<>();  
    private static Map<String, EventListener> eventListener = new HashMap<>();  
  
    public static void subscribeListener(String type, EventListener listener) {  
        eventListener.put(type, listener);  
    }  
  
    public static void publish(Event event) {  
        eventQueue.add(event);  
    }  
  
    public static void start() {  
  
        new Thread(() -> {  
  
            while (true) {  
  
                Event event = eventQueue.poll();  
  
                if (event != null) {  
  
                        EventListener listener = eventListener.get(event.getType());  
  
                    if (listener != null) {  
                        listener.onRecieve(event);  
                    }  
                }  
  
                try {  
                    Thread.sleep(10);  
                } catch (Exception e) {  
                    e.printStackTrace();  
                }  
  
                System.out.println("Pool Size: " + eventQueue.size());  
            }  
  
        }).start();  
  }  
}

```
**Evento:**
O evento contem um tipo, que nesta implementação usamos para identificar qual implementação de listener irá processá-lo, contem também uma informação que deverá ser computada, no nosso caso **payload** e também duas abstrações que representam callbacks que é um pedaço de código que é passado como parâmetro e que em algum momento será executado, no nosso caso na falha ou sucesso.
```java
package queuing.event;  
  
public class Event {  
  
    private String type;  
    private Object payload;  
    private Callback onSucess;  
    private Callback onFailure;  
  
    public Event(String type, Object payload, Callback onSucess, Callback onFailure) {  
        this.type = type;  
        this.payload = payload;  
        this.onSucess = onSucess;  
        this.onFailure = onFailure;  
    }  
  
    public String getType() {  
        return type;  
    }  
  
    public Object getPayload() {  
        return payload;  
    }  
  
    public Callback getOnSucess() {  
        return onSucess;  
    }  
  
    public Callback getOnFailure() {  
        return onFailure;  
    }  
}
```

**Callback:**
Recebe um argumento e deve processá-lo, nesta implementação definimos o conteúdo dos callbacks no **RegisterClientListener** e o seu comportamento no main, durante o momento da criação do evento.
```java
package queuing.event;  
  
public interface Callback {  
    void callback(Object payload);  
}
```

**Event Listener:**
Usado para realizar o polimorfismo no algoritmo do motor, assim nós podemos processar qualquer evento, desde que esteja mapeado.
```java
package queuing.event;  
  
public interface EventListener {
    void onRecieve(Event event);  
}

```
**Registro de Clientes Listener:**
Implementa a ação do evento, o que deve ser feito. Se houvessem novos eventos iriamos utilizar a interface **Event Listener** e implementar um comportamento para este novo evento.
```java
package queuing.event;  
  
import queuing.repository.Client;  
import queuing.repository.Repository;  
  
public class RegisterClientListener implements EventListener {  
  
    @Override  
    public void onRecieve(Event event) {  
  
        Repository repository = Repository.getInstance();  
  
        Client client = (Client) event.getPayload();  
        System.out.println("Initializing client registration: " + client);  
  
        Callback callback;  
  
		if (client.getId() % 2 != 0) {  
		    callback = event.getOnFailure();  
		    callback.callback(new RuntimeException("Invalid queuing.repository.Client!"));  
		}  
		else {  
		    repository.insertClient(client);  
		    callback = event.getOnSucess();  
		    callback.callback("Success on registration;");  
		}
    }  
}
```

Temos duas classes para representar os clientes a serem cadastrados e uma abstração de repositório onde vamos inserir os clientes que forem chegando.

**Cliente**
```java
package queuing.repository;  
  
import java.util.UUID;  
  
public class Client {  
  
    private Integer id;  
    private UUID name;  
  
    public Client(Integer id, UUID name) {  
        this.id = id;  
        this.name = name;  
    }  
  
    public Integer getId() {  
        return id;  
    }  
  
    @Override  
    public String toString() {  
        return "Client:{" +  
                "id=" + id +  
                ", name=" + name +  
                '}';  
    }  
}
```

**Repositório**
```java
package queuing.repository;  
  
import java.util.ArrayList;  
import java.util.List;  
  
public class Repository {  
  
    private List<Client> clientsRepository;  
  
    private static Repository instance = null;  
  
    public static Repository getInstance() {  
        if (instance == null) {  
            instance = new Repository();  
		}  
        return instance;  
    }  
  
    private Repository() {  
        this.clientsRepository = new ArrayList<>();  
    }  
  
    public void insertClient(Client client) {  
        this.clientsRepository.add(client);  
    }  
}
```

**Main:**
Por fim temos a classe Main, onde iniciamos o motor de evento e cadastramos um tipo de evento. Então realizamos o pedido de cadastro de 1 milhão de clientes, onde cada pedido é um evento que contem um tipo para ser identificado, um payload que é a informação a ser processada neste caso é o cliente, e então a implementação de dois callbacks que serão o que deve ser executado no caso de sucesso ou falha e que vai ser publicado na fila do motor para posteriormente ser consumido.
```java
package queuing;  
  
import queuing.event.Callback;  
import queuing.event.Event;  
import queuing.event.EventEngine;  
import queuing.event.RegisterClientListener;  
import queuing.repository.Client;  
  
import java.util.UUID;  
  
public class Main {  
  
    static {  
        EventEngine.start();  
        EventEngine.subscribeListener("CLIENT_REGISTRATION", new RegisterClientListener());  
      }  
  
    public static void main(String[] args) {  
  
        for (int i = 0; i < 1000000; i++) {  
            Client client = new Client(i, UUID.randomUUID());  
  
            Event event = new Event("CLIENT_REGISTRATION", client,
            new Callback() {  
                @Override  
                public void callback(Object payload) {  
                    String value = (String) payload;  
                    System.out.println("Success callback: " + value);  
                }  
            },  
            new Callback() {  
                @Override  
                public void callback(Object payload) {  
                    RuntimeException exception = (RuntimeException) payload;  
                    System.out.println("Failure callback: " + exception.getMessage());  
                }  
            });  
  
            EventEngine.publish(event);  
        }  
    }  
}
```
> **Observação**: É importante ser observado que por ter um motor de eventos que fica o tempo todo consumindo da fila, ao mesmo tempo em que os eventos estão sendo cadastrados estão também sendo consumidos.

Esse é um exemplo básico, mas dá para ter uma ideia do funcionamento deste paradigma.
