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

![Pipes](pipes.png){ width=800 }

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


###  Angular Arquitetura
Uma arquitetura escalável é aquela que pode crescer e se adaptar de maneira eficiente às mudanças no tamanho e na complexidade do sistema, sem comprometer o desempenho ou a manutenção. Para um aplicativo GUI, a escalabilidade refere-se à capacidade de lidar com:

Aumento do Tamanho dos Dados: A habilidade de carregar e manipular grandes volumes de dados sem prejudicar a performance.
Complexidade e Tamanho do Projeto: A capacidade de gerenciar projetos maiores e mais complexos, mantendo tempos de carregamento aceitáveis.
Além dos aspectos externos, a escalabilidade também envolve a facilidade de desenvolvimento e manutenção do aplicativo. Arquiteturas mal projetadas podem dificultar o desenvolvimento a longo prazo, levando a um aumento da complexidade, débito técnico e problemas de qualidade.

Benefícios de uma Arquitetura Bem Projetada
Embora uma boa arquitetura não elimine todos os problemas, ela proporciona uma base sólida para a equipe de desenvolvimento minimizar e resolver desafios. Em resumo, uma arquitetura bem projetada deve:

Desempenho Consistente: Manter uma boa experiência do usuário, independentemente do tamanho e da complexidade do aplicativo.
Facilidade de Desenvolvimento: Fornecer diretrizes claras e seguras para os desenvolvedores, mantendo a qualidade do projeto.
Simplicidade e Padrões de Design: Basear-se em padrões de design aceitos, reduzindo a curva de aprendizado.
Estrutura do Projeto
Para atingir esses objetivos, os módulos do aplicativo devem ser claramente organizados na estrutura de arquivos. Cada módulo deve estar em seu próprio diretório, contendo todos os arquivos relacionados (código, estilos, modelos, etc.). Um ponto crucial dessa abordagem é o isolamento dos módulos, garantindo que cada um seja independente e não faça referência a arquivos de outros módulos. Teoricamente, isso permite que qualquer módulo possa ser removido sem afetar o funcionamento do restante do aplicativo.


Angular possui uma arquitetura opinativa a respeito de modularização, ate o nest js copiou sua estrutrua solida de modularização.

Claro, no mundo real, não é possível seguir rigorosamente a regra de isolamento total dos módulos. Alguns serviços e componentes precisam ser reutilizados em toda a aplicação. Por isso, algumas funcionalidades são armazenadas em módulos "Core" e "Shared". A estrutura do aplicativo ficaria assim:

Módulos Específicos: Diretórios separados para cada módulo do aplicativo, cada um contendo seus próprios arquivos (código, estilos, modelos, etc.).

Módulo Core: Contém serviços e componentes fundamentais que são essenciais para o funcionamento da aplicação como um todo.

Módulo Shared: Armazena componentes, diretivas, pipes e outros recursos que podem ser compartilhados entre diferentes módulos do aplicativo.

#### Estrutura do Projeto

Core:
Serviços fundamentais
Componentes essenciais

Shared:
Componentes reutilizáveis
Diretivas
Pipes
Módulos Específicos:
Cada módulo com sua própria estrutura de arquivos


![Arquitetura exemplo](olympus.png){ width=800 }

###### Componentes de apresentação
Componentes de apresentação são componentes de UI que compõem os elementos visuais. Eles são componentes burros que aceitam dados externos e acionam eventos para ações como cliques em botões. Esses componentes personalizados são normalmente uma composição de elementos de UI, de bibliotecas semelhantes ng-bootstrapou Angular Materialcriadas para funcionalidade de negócios.

Podemos testar facilmente esses componentes do View para testar as ações e a visualização dos dados, uma vez que eles não têm nenhuma dependência direta com serviços ou estados externos.

são puramente interface de usuário e preocupados com a aparência das coisas.
não estão cientes da lógica de negócios ou serviços.
recebe dados via @Inputs e emite eventos via @Output.
###### Componentes do contêiner
Componentes de contêiner são os componentes que unem componentes de apresentação, serviços e gerenciamento de estado para fornecer funcionalidade de negócios.

Os componentes Container usam os serviços para buscar dados do backend e vinculá-los aos componentes de apresentação. Eles também ouvem os eventos provenientes dos componentes de apresentação e atualizam o estado de cada componente de apresentação e se comunicam com o backend.

Os componentes do contêiner são o centro de conexão das coisas, lidando com a lógica de negócios e delegando a apresentação aos componentes do View.

contém toda a lógica de negócios.
passar os dados para os componentes de apresentação e manipular eventos @Output gerados por eles.
não tem lógica de UI.
têm dependências de outras partes do seu aplicativo, como serviços ou seu armazenamento estatal.
###### Componentes da página
Componentes de página são os componentes que definem o layout de uma página com base em caminhos de URL. Por exemplo, você pode compor um componente de página com cabeçalho, rodapé e uma área no meio. Então, basicamente, você pode definir um componente Page para cada rota pai. Dentro de cada componente Page, você pode colocar seus componentes Container. Em alguns casos, você pode colocar componentes de apresentação diretamente dentro dos componentes de página, quando os componentes de visualização não precisam de nenhum comportamento dinâmico.

Ao usar componentes de página, isso ajudará os componentes do Container a saber menos sobre o layout fixo de uma página e a se concentrar mais no posicionamento dinâmico de componentes com base na lógica de negócios. Também ajudará a reduzir a profundidade dos componentes do Container para reutilizar o layout em diferentes rotas.

###### Arquitetura de fluxo de dados
Nas estruturas modernas de SPA, tudo é um componente. Eles são os principais blocos de construção para criar e controlar interfaces de usuário. Angular (e outras estruturas modernas) organizará os componentes em uma árvore hierárquica, o que significa que os componentes podem ter pai e filhos. Vamos imaginar que nossos componentes se comunicam entre si sem regras. Qualquer um deles estaria enviando dados e disparando eventos um para o outro e depois de um tempo, tudo ficaria muito confuso e estaríamos perdidos na floresta de solicitações e respostas de dados.

Com esse tipo de organização, precisamos garantir o fluxo de dados unidirecional dentro de nossos componentes pai e filho. A regra principal é que as ações aumentam e os fluxos de dados diminuem. Cada componente aceitará parâmetros @Input() para receber os dados de seu pai e poderá enviar o evento @Output() para notificar os assinantes de que algo aconteceu.


###### Componentes Inteligentes e Componentes Burros
Na nossa aplicação, introduzimos a ideia de componentes "inteligentes" e "burros". Os componentes inteligentes, também conhecidos como "Contêineres", são responsáveis pela lógica da aplicação, comunicação com serviços e efeitos colaterais, como chamadas de serviço e atualizações de estado. A ideia dessa divisão é separar claramente as responsabilidades:

Componentes Contêiner (Inteligentes)

```Typescript
import { ChangeDetectionStrategy, Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FacadeProductService } from '../../data-access/facades/facade-product.service';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { combineLatest, first } from 'rxjs';
import { FilterGenericComponent } from '../../../../shared/ui/filter-generic/filter-generic.component';
import { ProductTableComponent } from '../../ui/product-table/product-table.component';

@Component({
  selector: 'app-product-crud',
  standalone: true,
  imports: [CommonModule, FilterGenericComponent, ProductTableComponent],
  templateUrl: './product-crud.component.html',
  styleUrl: './product-crud.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductCrudComponent {

  readonly #logisticFacade = inject(FacadeProductService);
  readonly fields = this.#logisticFacade.fieldsFilter;
  readonly products = this.#logisticFacade.products;
  readonly fields$ = this.#logisticFacade
    .loadInitForms()
    .pipe(takeUntilDestroyed());
  readonly products$ = this.#logisticFacade
    .loadProducts()
    .pipe(takeUntilDestroyed());
  readonly vm$ = combineLatest([this.fields$, this.products$]);


  addingProduct() {
    this.#logisticFacade.createProduct();
  }

}
```


Os componentes contêiner são responsáveis por gerenciar a lógica da aplicação. Suas principais características são:

Lógica de Negócios: Contêm a lógica de negócios necessária para a aplicação.
Comunicação com Serviços: Realizam chamadas a serviços para buscar ou enviar dados.
Gerenciamento de Estado: Atualizam e mantêm o estado da aplicação.
Manipulação de Eventos: Recebem e manipulam eventos de outros componentes.
Efeitos Colaterais: Implementam ações que causam efeitos colaterais, como atualizações de estado e interações com o backend.

Componentes Burros
```Typescript
<ng-container>
  <mat-card>
    <mat-card-header>
      <mat-card-title>Tabela dos Produtos</mat-card-title>
      <button mat-raised-button color="primary" (click)="addingProduct()">
        Adicionar Produto
      </button>
    </mat-card-header>

    <mat-card-content>
      <app-generic-table
        [tableData]="products"
        [columns]="columns"
        [displayedColumns]="displayedColumns">
      </app-generic-table>
    </mat-card-content>
  </mat-card>
</ng-container>

import {
  ChangeDetectionStrategy,
  Component,
  EventEmitter,
  inject,
  Input,
  Output,
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { MatCardModule } from '@angular/material/card';
import { Product } from '../../models/model/product';
import { GenericTableComponent } from '../../../../shared/ui/generic-table/generic-table.component';
import { materialModules } from '../../../../shared/utils/material/material-module';
import { CategoryEnumTranslatePipe } from '../../../../shared/pipes/product/category-enum-translate.pipe';
import { ProductCategoryEnumTranslations } from '../../models/enum/product-category.enum';

@Component({
  selector: 'app-product-table',
  standalone: true,
  imports: [...materialModules, GenericTableComponent],
  templateUrl: './product-table.component.html',
  styleUrl: './product-table.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductTableComponent {
  @Input({
    required: true,
  })
  products!: Product[];

  @Output() addProduct=new EventEmitter<void>();

  columns = [
    {
      columnDef: 'id',
      header: 'Identificação do Produto',
      cell: (element: Product) => `${element.id}`,
    },
    {
      columnDef: 'name',
      header: 'Nome do Produto',
      cell: (element: Product) => `${element.name}`,
    },
    {
      columnDef: 'description',
      header: 'Descrição do Produto',
      cell: (element: Product) => `${element.description}`,
    },
    {
      columnDef: 'category',
      header: 'Categoria de Produto',
      cell: (element: Product) => `${this.translateCategory(element)}`,
    },
    {
      columnDef: 'height',
      header: 'Altura do Produto',
      cell: (element: Product) => `${element.height} `,
    },
    {
      columnDef: 'width',
      header: 'Largura do Produto',
      cell: (element: Product) => `${element.width}`,
    },
    {
      columnDef: 'storageInstructions',
      header: 'Instruções de Armazenamento do Produto',
      cell: (element: Product) => `${element.storageInstructions}`,
    },
    {
      columnDef: 'restrictions',
      header: 'Restrições do Produto',
      cell: (element: Product) => `${element.restrictions}`,
    },
  ];

  translateCategory(product: Product, ...args: unknown[]): string {
    const category = product.category;
    return ProductCategoryEnumTranslations[category] || 'Unknown Category';
  }

  displayedColumns = this.columns.map(c => c.columnDef);

  addingProduct()
  {
    this.addProduct.emit();
  }
}
```



Os componentes burros são focados na apresentação e interação com o usuário. Suas principais características são:

Foco na UI: Responsáveis apenas pela renderização visual e interação com o usuário.
Recebimento de Dados: Obtêm todos os dados necessários através de propriedades @Input.
Emissão de Eventos: Comunicam-se com o exterior emitindo eventos via @Output.
Pouca ou Nenhuma Lógica: Contêm pouca ou nenhuma lógica de negócios, sendo principalmente componentes de apresentação.
Benefícios da Divisão
A divisão entre componentes contêineres e burros traz vários benefícios para a aplicação:

Clareza e Manutenção: Facilita a manutenção do código, separando claramente as responsabilidades.
Reutilização: Componentes burros podem ser facilmente reutilizados em diferentes partes da aplicação, pois são independentes da lógica de negócios.
Testabilidade: Componentes burros são mais fáceis de testar, pois não dependem de serviços ou estados externos.
Organização: Ajuda a organizar a aplicação de maneira modular e escalável.


### Cancelar assinatura de Observáveis

#### Usar pipe async 

O pipe assíncrono assina um Observable ou Promise e retorna o valor mais recente que emitiu. 
Quando um novo valor é emitido, o pipe assíncrono marca o componente a ser verificado quanto a alterações. Quando o componente é destruído, o asyncpipe cancela a assinatura automaticamente para evitar possíveis vazamentos de memória. 
Usando-o em nosso AppComponent:
```Typescript
@Component({
    ...,
    template: `
        <div>
         Interval: {{observable$ | async}}
        </div>
    `
})
export class AppComponent implements OnInit {
    observable$
    ngOnInit () {
        this.observable$ = Rx.Observable.interval(1000);
    }
}

```

Na instanciação, o AppComponent criará um Observable a partir do método interval. No modelo, o Observable observable é canalizado para o Pipe assíncrono.

O pipe assíncrono assinará o observável e exibirá seu valor no DOM. o pipe assíncrono cancelará a assinatura do observável$ quando o AppComponent for destruído. async Pipe possui ngOnDestroy em sua classe, portanto, é chamado quando a visualização contida está sendo destruída. Usar o pipe async é uma grande vantagem se estivermos usando Observables em nossos componentes, porque ele irá assiná-los e cancelar a assinatura deles. Não nos preocuparemos em esquecer de cancelar a assinatura deles no ngOnDestroy quando o componente estiver sendo eliminado.

#### Cancelamento de inscrição declarativamente com takeUntil
A solução é compor nossas assinaturas com o operador takeUntil e usar um assunto que emita um valor verdadeiro no gancho do ciclo de vida ngOnDestroy.

O trecho a seguir faz exatamente a mesma coisa, mas desta vez cancelamos a assinatura declarativamente. Você notará que um benefício adicional é que não precisamos mais manter referências às nossas assinaturas:

```Typescript

import { Observable } from 'rxjs/Observable';
import { Subject } from 'rxjs/Subject';
import 'rxjs/add/observable/interval';
import 'rxjs/add/operator/takeUntil';

@Component({ ... })
export class AppComponent implements OnInit, OnDestroy {
  destroy$: Subject<boolean> = new Subject<boolean>();

  constructor() {}

  ngOnInit(){
    Observable
      .interval(250)
      .takeUntil(this.destroy$)
      .subscribe(val => {
        console.log('Current value:', val);
      });
  }

  ngOnDestroy() {
    this.destroy$.next(true);
    this.destroy$.unsubscribe();
  }

```