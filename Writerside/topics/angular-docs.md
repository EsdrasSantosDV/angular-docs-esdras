# Angular Docs
![Create new topic options](angular.png){ width=800 }

<!--Writerside adds this topic when you create a new documentation project.
You can use it as a sandbox to play with Writerside features, and remove it from the TOC when you don't need it anymore.-->

## Otimizando a renderização de listas no Angular com trackBy

Vamos começar entendendo a importância do trackBy no Angular. 
Quando usamos a diretiva ngFor para renderizar uma lista de itens,
o Angular rastreia as alterações na lista utilizando a identidade de cada item.
Normalmente, o Angular se baseia em referências de objetos para identificar esses itens.
Isso significa que, se você modificar um objeto dentro da lista, o Angular irá re-renderizar a lista inteira, 
mesmo que apenas um único objeto tenha sido alterado.

### Problemas com Grandes Listas
Isso pode ser um problema se você tiver uma lista grande, 
pois renderizar novamente a lista inteira pode ser caro em termos de desempenho.
A manipulação excessiva do DOM pode causar lentidão e degradação na experiência do usuário.
Suponha que temos alguns dados provenientes de alguma API de back-end e 
estamos armazenando dados em algum tipo de coleção, como um array, 
e então precisamos atualizar esses dados na página da web usando a diretiva ngFor.
Por padrão, o que a estrutura angular fará é remover todos os elementos DOM que estão associados aos dados e criá-los novamente na árvore DOM, mesmo que dados iguais estejam chegando.


### Implementando Track By em Listas Grandes no Angular
Ao implementar o trackBy no Angular, é essencial usar um identificador exclusivo
para cada item da lista. 
Esse identificador exclusivo é crucial para que o Angular rastreie as alterações na
lista de forma precisa. Sem um identificador exclusivo, 
o Angular pode não conseguir identificar corretamente quais itens foram adicionados, removidos ou atualizados, resultando em problemas de renderização e desempenho ineficientes. Isso é especialmente importante em listas dinâmicas onde os itens são frequentemente atualizados ou reordenados.

Vamos entender como isso funciona no contexto de uma lista de personagens de One Piece, considerando um cenário onde temos milhões de personagens.

```Typescript
import { Component } from "@angular/core";

@Component({
  selector: "app-characters-list",
  template: `
    <ul>
      <li *ngFor="let character of characters; trackBy: identify">{{ character.name }}</li>
    </ul>
  `,
})
export class CharactersListComponent {
  characters: any[] = [
    { id: 1, name: "Luffy" },
    { id: 2, name: "Zoro" },
    { id: 3, name: "Nami" },
    { id: 4, name: "Sanji" },
  ];

  public identify(index: number, character: any) {
    return character.id; // identificador único do personagem
  }
}
```

## Crime Brutal: O Erro de Chamar Métodos no Template em Angular e Como Evitá-lo

### Meu Relato de Estagiário
Quando eu era estagiário, sempre que precisava pegar um tipo de dado e realizar
uma transformação para uma determinada formatação ou até para transformá-lo
dependendo de uma condição, cometia um erro clássico:
chamava algum método diretamente no template, e dentro desse método realizava a transformação. De todos os projetos que analisei até hoje, esse é um dos erros mais comuns que vi sendo cometido. E posso garantir que isso já gerou uma série de problemas de desempenho.

Estava eu lá, todo empolgado com Angular, quando percebia que precisava transformar um dado antes de exibi-lo. Sem pensar duas vezes, lá ia eu:

```Typescript
<td mat-cell *matCellDef="let element" class="mat-cell-text-to-left">
  {{ displayOrderUnit(element?.orderUnit) }}
</td>
displayOrderUnit(orderUnit: CommodityOrderUnitEnum) {
  return orderUnit ? OrderUnitNamesRecord[orderUnit] : '';
}
```
Parece simples e inofensivo, né? Mas a verdade é que isso causava um impacto brutal no desempenho da aplicação.
O problema está no sistema de detecção de alterações do Angular. Vamos entender por quê.


O Problema com Métodos no Template
O objetivo da detecção de alterações do Angular é
descobrir quais partes da interface do usuário precisam ser renderizadas novamente quando ocorrem alterações.
Para determinar se <td mat-cell *matCellDef="let element" class="mat-cell-text-to-left">displayOrderUnit(element?.orderUnit) </td>
precisa ser renderizado novamente, o Angular precisa executar a expressão displayOrderUnit() para verificar se seu valor de retorno mudou.

Como o Angular não pode prever se o valor de retorno da função mudou, ele precisa executar
a função toda vez que a detecção de alterações é executada. Portanto,
se a detecção de alterações for executada 1000 vezes,
a função será chamada 1000 vezes, mesmo que seu valor de retorno nunca mude. Dependendo da lógica dentro da função,
executar uma função centenas de vezes pode se tornar um sério problema de desempenho.


A boa notícia é que, nas versões mais recentes do Angular,
os Signals ajudam a mitigar esse problema, mas vamos falar sobre isso mais tarde.
Nos casos em que o estado não muda, os pipes são uma excelente solução.

### Introdução aos Pipes
Em Angular, pipes são uma ferramenta poderosa que permite transformar dados antes de exibi-los em seus modelos.
Pipes são uma forma de formatar, filtrar e manipular dados sem modificar a fonte de dados subjacente.
Angular fornece duas categorias principais de pipes: pipes puros e pipes impuros.

```Typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'orderUnit',
  pure: true, // pipe puro POR PADRÃO E PURO
  standalone: true,
})
export class OrderUnitPipe implements PipeTransform {
  transform(orderUnit: CommodityOrderUnitEnum): string {
    return orderUnit ? OrderUnitNamesRecord[orderUnit] : '';
  }
}
```

Pipes Puros
Os pipes puros são projetados para serem sem estado e determinísticos.
Isso significa que sua saída depende exclusivamente da sua entrada e não tem quaisquer efeitos colaterais.
Eles devem ser usados com dados imutáveis, onde os dados de entrada permanecem inalterados durante a transformação do pipe.
Angular otimiza pipes puros recalculando sua saída apenas se os dados de entrada mudarem.

Pipes Impuros

```Typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'isToday',
  pure: false
})
export class IsTodayPipe implements PipeTransform {
  transform(value: Date): string {
    const today = new Date();
    if (value.getDate() === today.getDate() &&
        value.getMonth() === today.getMonth() &&
        value.getFullYear() === today.getFullYear()) {
      return 'Today';
    } else {
      return 'Not Today';
    }
  }
}
```

Os pipes impuros, por outro lado, podem ter efeitos colaterais e podem depender do estado externo.
Eles podem recalcular sua saída mesmo que os dados de entrada permaneçam os mesmos.
Isso pode acontecer, por exemplo, quando o pipe impuro depende de dados assíncronos ou de mudanças de estado global.


Chamar métodos diretamente no template é um crime brutal para o desempenho da sua aplicação Angular.
Em vez disso, use pipes para transformar seus dados de forma eficiente.
Lembre-se, os pipes puros são ideais para dados imutáveis e garantem que as transformações sejam feitas de forma otimizada.
Para casos mais complexos, onde o estado pode mudar ou depender de dados externos, os pipes impuros podem ser utilizados.
E assim, evitamos a re-renderização desnecessária, mantemos nossa aplicação performática e, claro, nos livramos do pesadelo de ver nossos métodos sendo chamados centenas de vezes sem necessidade. Viva os pipes!



### Realizar Request HTTP no Angular: HttpClient
Angular é uma framework bastante opinativo, e isso é uma das coisas que o torna tão poderoso.
Ele fornece uma série de recursos e ferramentas para facilitar o desenvolvimento de aplicativos web modernos.
Um de seus principais recursos é a capacidade de fazer solicitações HTTP facilmente, 
permitindo recuperar dados de APIs e interagir com serviços de back-end.
No entanto, enviar e manipular solicitações 
HTTP de forma eficiente é crucial para o desempenho geral do seu aplicativo Angular.

Para fazer solicitações HTTP no Angular, recomenda-se fortemente o uso do módulo HttpClient, que é integrado à plataforma. O HttpClient oferece uma API simples, mas poderosa, para enviar solicitações e lidar com respostas. Ele gerencia automaticamente detalhes como cabeçalhos de solicitação, tokens CSRF e a análise de respostas. Além disso, o HttpClient suporta observáveis, o que facilita o tratamento de operações assíncronas e a execução de operações avançadas, como tratamento de erros e cancelamento.


#### Configuração do HttpClient
```Typescript
//NA VERSÃO 18 DO ANGULAR O HttpClientModule foi removido
import {
  provideHttpClient,
} from '@angular/common/http';
    provideHttpClient(),
    
VERSÃO ANTERIORES E QUE USAM UMA ESTRUTURA DE PROJETO DE MODULOS
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  imports: [
    HttpClientModule,
  ],
})
```
O serviço que criamos no Angular é responsável por gerenciar um crud de personagens de dragon ball.
Ele utiliza o HttpClient para fazer requisições HTTP para uma API e realizar operações como listar, 
criar, atualizar e deletar personagens. Vamos detalhar cada parte de forma simples.

Primeiro, importamos os módulos necessários e criamos o serviço com o decorator @Injectable, 
permitindo que ele seja injetado em outros componentes ou serviços.

```Typescript
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable, shareReplay } from 'rxjs';
import { Character } from '../interfaces/character';
import { ICreateCharacterDto } from '../dto/create-character.dto';
import { IUpdateCharacterDto } from '../dto/update-character.dto';
import { GetAllCharactersDto } from '../dto/get-all-characters.dto';

@Injectable({
  providedIn: 'root',
})
export class CharacterService {
  constructor(private readonly http: HttpClient) {}

  baseUrl = 'api/dragonball-characters';

  //Listar Personagens Paginados

  //Este método obtém uma lista paginada de personagens, permitindo filtros por nome e raça. Ele monta os parâmetros da requisição com base nos valores fornecidos.
  getCharactersPaginated(
    getAllCharactersDto: GetAllCharactersDto
  ): Observable<{ items: Character[]; count: number }> {
    const { page, limit, name, race } = getAllCharactersDto;
    let params = new HttpParams();

    if (page) {
      params = params.set('page', page.toString());
    }

    if (limit) {
      params = params.set('limit', limit.toString());
    }

    return this.http.get<{ items: Character[]; count: number }>(this.baseUrl, {
      params,
    });
  }
  //Listar Todos os Personagens
  //Este método retorna todos os personagens sem paginação
  getAllCharacters(): Observable<Character[]> {
    return this.http.get<Character[]>(this.baseUrl + '/all');
  }

  //O GENERICO E O VALOR DE RETORNO DO METODO
  createCharacter(createCharacterDto: ICreateCharacterDto): Observable<Character> {
    return this.http.post<Character>(this.baseUrl, createCharacterDto);
  }
  
  updateCharacter(updateCharacterDto: IUpdateCharacterDto): Observable<void> {
    const { id, ...updatedCharacterDtoWithoutId } = updateCharacterDto;
    return this.http.patch<void>(
      this.baseUrl + `/${updateCharacterDto.id}`,
      updatedCharacterDtoWithoutId
    );
  }

  deleteCharacter(id: number): Observable<void> {
    return this.http.delete<void>(this.baseUrl + `/${id}`);
  }

  enableOrDisableCharacter(id: number): Observable<void> {
    return this.http.put<void>(this.baseUrl + `/${id}`, {});
  }
}
```

Os observáveis desempenham um papel crucial no modelo de programação reativa do Angular. Ao fazer solicitações HTTP, é essencial aproveitar o poder dos observáveis. Eles permitem lidar com operações assíncronas e gerenciar fluxos de dados de forma eficiente. Você pode usar operadores como map, filter, e merge para transformar e combinar observáveis. Além disso, os observáveis fornecem recursos como retry e catchError, que podem ser úteis para lidar com cenários de erro ou limitação de taxa.

