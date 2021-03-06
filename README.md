# CThreads
__Autores:__ Béuren F. Bechlin, Eduardo Brito, Levindo Neto

![States](/resources/States.png)

### Estrutura dos Estados

A fila de aptos é modelada como uma árvore rubro-negra, uma vez que como é usado o 
sorteio para a seleção de qual será o processo usado, não tem como ser FIFO ou SJF.
Para os bloqueados, é uma árvore rubro-negra para aqueles que foram bloqueados pelo semáforo. 

Também há uma rubro-negra para processos bloqueados por ter dado cjoin, 
que ficam bloqueados esperando a thread. Quando as threads são finalizadas
é necessário fazer a varredura para verificar se algum bloqueado pode passar para apto. 

### Sorteio de bilhetes

É implementado uma lista duplamente encadeada, na qual, dado um número 
inteiro de bilhete (0-255), percorre-se essa lista sempre salvando o elemento 
anterior na tentativa de encontrar o número referente ao bilhete, se chegar em 
um valor maior que o número de bilhete pesquisado, compara-se a diferença desse
valor com o inteiro do bilhete com a diferença entre o valor anterior e o bilhete.
A thread que tiver a menor diferença sai do estado de apto e vai para o executando.

### Design Top-Down
![TOP-DOWN](/resources/Top-Down_Diagram.png)

### Funções da API

#### CCreate
```c
int ccreate (void *(*start)(void *), void *arg); 
```

Cria uma TCB, usando o valor TID. Para saber o valor do TID usado terá uma variável 
de controle que é incrementada a cada criação de thread, que será usada para o TID, 
garantindo que ele é sempre diferente. Também colocamos o valor em ticket para o valor 
randômico gerado usando a função passada pelo professor. O estado até poderia começar
em 0 para indicar a criação, mas como após criado vai para 1, indicando que está apto, 
é colocado diretamente 1.

__Parâmetros:__ 
* start: ponteiro para a função que a thread executará. 
* arg: um parâmetro que pode ser passado para a thread na sua criação. 

(Obs.: é um único parâmetro. Se for necessário passar mais de um valor deve-se empregar um ponteiro para uma struct) 

__Retorno:__ 
Quando executada corretamente retorna um valor positivo, que representa o 
identificador da thread criada. Caso contrário, retorna um valor negativo.

```c
typedef struct s_TCB { 

int tid; // identificador da thread int state; // estado em que a thread se encontra // 0: Criação; 1: Apto; 2: Execução; 3: Bloqueado e 4: Término 
int ticket; // “bilhete” de loteria da thread, para uso do dispatcher 
ucontext_t context; // contexto de execução da thread (SP, PC, GPRs e recursos) 

} TCB_t;
```

#### CYield

```c
int cyield(void); 
```

Nesse comando damos Yield(), além disso setamos o estado pro valor 1, pra informar que está na fila de aptos e devemos também salvar o contexto, colocando ele na variável context da TCB.

Após isso procura na ED  se um processo está bloqueado esperando por ele, se sim então desbloqueia esse processo. Então é chamado o dispatcher de threads.

Retorno: Quando executada corretamente: retorna 0 (zero) Caso contrário, retorna um valor negativo.

#### CJoin

```c
int cjoin(int tid);
```

Essa função vai criar uma fila a qual possui a informação das threads que estão sendo esperadas serem finalizadas. Dessa forma, o processo que deu cjoin é colocado como bloqueado, e toda vez que um processo for finalizado com sucesso deve-se consultar essa fila e verificar se era um processo que alguém estava esperando. Se sim, então a partir da informação da fila, percorre-se os bloqueados e coloca ele novamente para apto.


__Parâmetros:__ TID: identificador da thread cujo término está sendo aguardado. 

__Retorno:__ Quando executada corretamente: retorna 0 (zero) Caso contrário, retorna um valor negativo.

```c
typedef struct s_sem { 
int count; // indica se recurso está ocupado ou não (livre > 0, ocupado ≤ 0)
PFILA2 fila; // ponteiro para uma fila de threads bloqueadas no semáforo. 
} csem_t;
```


Criaremos uma fila FIFO utilizada para as threads bloqueadas no semáforo. FIFO pois é relativamente simples de implementar, feita como uma lista encadeada simples.

```c
int csem_init (csem_t *sem, int count);
```


Inicializa variável do semáforo em 1, utiliza o count (inteiro que mostra a quantidade de recursos que o semáforo está controlando). O valor inicial do semáforo tem que ser 1 para se ter exclusão mútua. A variável sem deve ter junto a ela uma estrutura com o registro das threads que estão bloqueadas. Essa lista inicia vazia.

__Parâmetros:__ sem: ponteiro para uma variável do tipo csem_t. Aponta para uma estrutura de dados que representa a variável semáforo. 
count: valor a ser usado na inicialização do semáforo. Representa a quantidade de recursos controlados pelo semáforo.

__Retorno:__ Quando executada corretamente: retorna 0 (zero) Caso contrário, retorna um valor negativo.

#### CSem_init

```c
int csem_init (csem_t *sem, int count);
```

Faz a inicialização do semáforo colocando csem_t=1.
Há uma ligação entre as variáveis de semáforo e suas respectivas threads bloqueadas esperando liberação.

__Parâmetros:__
sem: ponteiro para uma variável do tipo csem_t. Aponta para uma estrutura de dados que representa a variável semáforo.
count: valor a ser usado na inicialização do semáforo. Representa a quantidade de recursos controlados pelo semáforo.

__Retorno:__
Quando executada corretamente: retorna 0 (zero). Caso contrário, retorna um valor negativo


#### CWait

```c
int cwait (csem_t *sem);
```

Teremos um valor de count para verificar quando a thread deve ser colocada em “bloqueado” e pra colocar na fila de bloqueados do semáforo quando ela for <=0. Chamou cwait faz count--;

A variável de semáforo (sem) monitora os recursos. Sempre é verificado se há recurso livre, se não a thread é bloqueada, essa verificação é feita com um comparador com <0, a cada recurso alocado decrementa, e assim que o recurso for desalocado incrementa a variável, assim pode-se alocar um recurso a outro processo.

__Parâmetros:__ Ponteiro pra variável de semáforo e o count.

#### CSignal

```c
int csignal (csem_t *sem);
```

Como precisa de algo pra avisar que a thread que estava usando um recurso parou de usar, a csignal faz isso, daí apenas incrementa a variável de semáforo (V(s)), daí verifica na fila de bloqueados do semáforo quem estava esperando pelo recurso e libera pro uso.
__Parâmetros:__  Apenas um ponteiro para a variável de semáforo.

#### CIdentify

```c
int cidentify (char *name, int size); 
```


Basicamente aponta para a string que haverá as informações do grupo que serão impressas via printf. Corta de acordo com o size.

__Parâmetros:__ name: ponteiro para uma área de memória onde deve ser escrito um string que contém os nomes dos componentes do grupo e seus números de cartão. Deve ser uma linha por componente. size: quantidade máxima de caracteres que podem ser copiados para o string de identificação dos componentes do grupo.

__Retorno:__ Quando executada corretamente: retorna 0 (zero) Caso contrário, retorna um valor negativo


### Funções internas

#### dispatcher

```c
void dispatcher(void);
```


Sorteia o ticket para selecionar qual a próxima thread que deve ir para o estado executando e então procura na fila de apto por quem tem esse ticket ou está mais perto do número dele. Então, faz o chaveamento de contexto dessa thread alterando o seu estado para executando. 

Responsável também por fazer a varredura dos bloqueados para verificar se a thread que recém finalizou e chamou o dispatcher libera algum desses processos, que passará para a fila de aptos.
