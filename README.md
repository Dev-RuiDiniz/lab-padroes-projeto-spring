Explorando Padrões de Projetos na Prática com Java ☕
Este repositório foi criado para explorar e exemplificar a aplicação de alguns dos principais padrões de projeto em um projeto Java, com o apoio do Spring Framework. O objetivo é demonstrar como esses padrões podem ser utilizados para criar sistemas mais robustos, flexíveis e de fácil manutenção.

Os padrões abordados são:

Singleton

Strategy/Repository

Facade

A seguir, uma breve explicação e exemplos de como cada um desses padrões pode ser aplicado.

1. Padrão Singleton
O padrão Singleton garante que uma classe tenha apenas uma única instância e oferece um ponto de acesso global a ela. Ele é ideal para recursos que precisam ser compartilhados em toda a aplicação, como um gerenciador de configurações, um pool de conexões com banco de dados ou um serviço sem estado específico.

No contexto do Spring Framework, a implementação do Singleton é simplificada e gerenciada por padrão. Todos os beans (componentes gerenciados pelo Spring) são singletons, a menos que uma outra scope seja explicitamente definida (ex: prototype, request, session).

Exemplo Prático:

Uma classe anotada com @Service como ClienteService é automaticamente gerenciada como um singleton pelo Spring.

// src/main/java/com/example/designpatterns/service/ClienteService.java
package com.example.designpatterns.service;

import org.springframework.stereotype.Service;
import com.example.designpatterns.model.Cliente;

@Service // O Spring cria e gerencia uma única instância desta classe
public class ClienteService {

    public Cliente findById(Long id) {
        // Lógica para buscar um cliente no banco de dados.
        // Neste exemplo, simulamos a busca.
        System.out.println("DEBUG: Buscando cliente com ID: " + id + " (usando o ClienteService Singleton)");
        return new Cliente(id, "João da Silva - ID: " + id);
    }

    public void save(Cliente cliente) {
        // Lógica para salvar um cliente no banco de dados.
        System.out.println("DEBUG: Salvando cliente: " + cliente.getNome() + " (usando o ClienteService Singleton)");
    }
}

O Spring cuida da criação e gerenciamento de uma única instância de ClienteService, garantindo que ela seja reutilizada em toda a aplicação sempre que for injetada.

2. Padrão Strategy / Repository
O padrão Strategy define uma família de algoritmos, encapsula cada um deles e os torna intercambiáveis. Ele permite que o algoritmo varie independentemente dos clientes que o utilizam. O padrão Repository é frequentemente usado em conjunto com o Strategy, pois ele abstrai a lógica de acesso aos dados, permitindo que você mude a forma como os dados são acessados (por exemplo, de um banco de dados para um serviço externo) sem alterar a lógica de negócio principal.

Exemplo Prático:

Vamos supor que você precise buscar dados de endereço, e essa busca pode ser feita de diferentes maneiras ou por diferentes fontes (ex: banco de dados local, API externa).

Interface da Estratégia (Repository):
Define o contrato para a operação de acesso a dados.

// src/main/java/com/example/designpatterns/repository/EnderecoRepository.java
package com.example.designpatterns.repository;

import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;
import com.example.designpatterns.model.Endereco;

@Repository // Define esta interface como um componente de repositório do Spring Data
public interface EnderecoRepository extends CrudRepository<Endereco, String> {
    // Endereços são tipicamente identificados por um CEP, por isso a chave é uma String.
    // O Spring Data JPA (se configurado) automaticamente implementa os métodos
    // CRUD (Create, Read, Update, Delete) para você, agindo como nossa "estratégia" de persistência.
}

Contexto (Serviço que usa a Estratégia):
Um serviço que utiliza a estratégia (neste caso, o EnderecoRepository) sem se preocupar com os detalhes da implementação.

// src/main/java/com/example/designpatterns/service/ClienteService.java (atualizado)
package com.example.designpatterns.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import com.example.designpatterns.model.Cliente;
import com.example.designpatterns.model.Endereco;
import com.example.designpatterns.repository.EnderecoRepository; // Importa o EnderecoRepository

@Service
public class ClienteService {

    @Autowired // Injeta a instância do EnderecoRepository
    private EnderecoRepository enderecoRepository;

    // ... métodos existentes ...

    public Endereco buscarEnderecoPorCep(String cep) {
        // A lógica de negócio usa a estratégia (repository) para buscar o endereço.
        // O ClienteService não se importa de onde o endereço vem, apenas que ele pode ser obtido.
        System.out.println("DEBUG: Buscando endereço para o CEP: " + cep + " (usando EnderecoRepository)");
        return enderecoRepository.findById(cep)
                .orElseThrow(() -> new IllegalArgumentException("Endereço não encontrado para o CEP: " + cep));
    }
}

Dessa forma, o ClienteService não precisa saber como o endereço é buscado (se é de um banco de dados, de um arquivo ou de uma API). Ele apenas usa a interface EnderecoRepository, e o Spring injeta a implementação correta. Isso desacopla a lógica de negócio da lógica de acesso a dados, tornando o sistema mais flexível.

3. Padrão Facade
O padrão Facade (Fachada) fornece uma interface unificada e simplificada para um conjunto de interfaces em um subsistema. Ele define uma interface de nível superior que torna o subsistema mais fácil de usar, escondendo a complexidade interna de suas operações.

Exemplo Prático:

Imagine um processo complexo, como a realização de um pedido de compra, que envolve várias etapas e interações com diferentes serviços (buscar cliente, buscar produtos, calcular frete, etc.). Em vez de o cliente do seu código ter que coordenar todas essas chamadas, você pode criar uma "fachada" para simplificar o processo.

// src/main/java/com/example/designpatterns/facade/PedidoServiceFacade.java
package com.example.designpatterns.facade;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import com.example.designpatterns.model.Cliente;
import com.example.designpatterns.model.Produto;
import com.example.designpatterns.model.Pedido;
import com.example.designpatterns.service.ClienteService;
import com.example.designpatterns.service.ProdutoService;
import com.example.designpatterns.service.FreteService;

import java.util.List;

@Service // A fachada também é um serviço Spring
public class PedidoServiceFacade {

    @Autowired
    private ClienteService clienteService; // Sub-sistema 1
    @Autowired
    private ProdutoService produtoService; // Sub-sistema 2
    @Autowired
    private FreteService freteService;     // Sub-sistema 3

    public Pedido realizarPedido(Long clienteId, List<Long> produtoIds) {
        System.out.println("\nDEBUG: Iniciando processo de realização de pedido (via Facade)");

        // 1. Busca o cliente
        Cliente cliente = clienteService.findById(clienteId);
        if (cliente == null) {
            throw new IllegalArgumentException("Cliente não encontrado com o ID: " + clienteId);
        }
        System.out.println("DEBUG: Cliente encontrado: " + cliente.getNome());

        // 2. Cria o pedido e adiciona os produtos
        Pedido pedido = new Pedido(cliente);
        for (Long produtoId : produtoIds) {
            Produto produto = produtoService.findById(produtoId);
            if (produto == null) {
                System.out.println("AVISO: Produto com ID " + produtoId + " não encontrado e foi ignorado.");
            } else {
                pedido.adicionarItem(produto);
                System.out.println("DEBUG: Produto adicionado ao pedido: " + produto.getNome());
            }
        }

        // 3. Calcula o frete (assumindo que o cliente tem um endereço para o cálculo)
        double frete = 0.0;
        if (cliente.getEndereco() != null && cliente.getEndereco().getCep() != null) {
            frete = freteService.calcularFrete(cliente.getEndereco().getCep());
            pedido.setFrete(frete);
            System.out.println("DEBUG: Frete calculado: R$" + String.format("%.2f", frete));
        } else {
            System.out.println("AVISO: Não foi possível calcular o frete, endereço do cliente ausente.");
        }

        // 4. Salva o pedido e finaliza o processo (lógica simplificada)
        System.out.println("DEBUG: Pedido finalizado para o cliente: " + cliente.getNome() +
                           " | Total de itens: " + pedido.getItens().size() +
                           " | Frete: R$" + String.format("%.2f", pedido.getFrete()));

        // Em uma aplicação real, você persistiria o pedido aqui.
        return pedido;
    }
}

Neste exemplo, o PedidoServiceFacade atua como uma fachada para o complexo processo de compra. Ele encapsula as chamadas para ClienteService, ProdutoService e FreteService, expondo uma única e simples interface (realizarPedido) para o cliente do código. Isso torna o código mais limpo, mais fácil de entender e menos propenso a erros, pois a complexidade é "escondida" por trás da fachada.

