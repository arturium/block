# Building Blockchain in Go. Part 3: Persistence and CLI

## Вступление
Пока что мы создали блокчейн с системой проверки работоспособности, которая делает возможным майнинг. Наша реализация приближается к полнофункциональной цепочке блоков, но ей все еще не хватает некоторых важных функций. Сегодня начнется хранение блокчейна в базе данных, и после этого мы сделаем простой интерфейс командной строки для выполнения операций с блокчейном. По своей сути блокчейн представляет собой распределенную базу данных. Сейчас мы собираемся опустить «распределенную» часть и сосредоточимся на части «база данных».

## Выбор базы данных
В настоящее время в нашей реализации нет базы данных; вместо этого мы создаем блоки каждый раз, когда запускаем программу, и сохраняем их в памяти. Мы не можем повторно использовать блокчейн, мы не можем поделиться им с другими, поэтому нам нужно сохранить его на диске.

Какая база данных нам нужна? На самом деле, любой из них. В оригинальной статье о биткойнах ничего не говорится об использовании определенной базы данных, так что решать, какую базу данных использовать, зависит от разработчика. Bitcoin Core , который был первоначально опубликован Satoshi Nakamoto и который в настоящее время является эталонной реализацией Bitcoin, использует LevelDB (хотя он был представлен клиенту только в 2012 году). И мы будем использовать ...

## BoltDB
Так как:

Это просто и минималистично.  
Это реализовано в Go.   
Не требует запуска сервера.   
Это позволяет построить структуру данных, которую мы хотим.  

Из ЧИТАНИЯ BoltDB на Github :

***Bolt*** - это магазин ключей и ценностей Go, вдохновленный проектом LMDB Говарда Чу. Цель проекта - предоставить простую, быструю и надежную базу данных для проектов, которым не требуется полноценный сервер баз данных, такой как Postgres или MySQL.

Поскольку Bolt предназначен для использования в качестве такого низкого уровня функциональности, простота является ключевым фактором. API будет небольшим и сосредоточится только на получении значений и настройке значений. Вот и все.

Звучит идеально для наших нужд! Давайте потратим минуту на его рассмотрение.

BoltDB - это хранилище ключей / значений, что означает отсутствие таблиц, подобных SQL RDBMS (MySQL, PostgreSQL и т. Д.), Ни строк, ни столбцов. Вместо этого данные хранятся в виде пар ключ-значение (как в картах Голанга). Пары ключ-значение хранятся в сегментах, которые предназначены для группировки похожих пар (это похоже на таблицы в RDBMS). Таким образом, чтобы получить значение, вам нужно знать ведро и ключ.

Важной особенностью BoltDB является отсутствие типов данных: ключи и значения являются байтовыми массивами. Так как мы будем хранить в нем структуры Go ( Blockв частности), нам нужно будет их сериализовать, то есть реализовать механизм преобразования структуры Go в байтовый массив и его восстановления из байтового массива. Мы будем использовать кодирование / комок для этого, но JSON, XML, Protocol Buffersи т.д. могут быть использованы. Мы используем, encoding/gobпотому что это просто и является частью стандартной библиотеки Go.

## Структура базы данных
Прежде чем приступить к реализации логики персистентности, нам нужно сначала решить, как мы будем хранить данные в БД. 
И для этого мы будем ссылаться на то, как это делает Bitcoin Core.

Проще говоря, Bitcoin Core использует два «контейнера» для хранения данных:

blocks хранит метаданные, описывающие все блоки в цепочке.
chainstate хранит состояние цепочки, которая представляет собой все неизрасходованные на данный момент выходные данные транзакции и некоторые метаданные.
Также блоки хранятся в виде отдельных файлов на диске. Это сделано с целью повышения производительности: чтение одного блока не потребует загрузки всех (или некоторых) из них в память. Мы не будем это реализовывать.

```
В blocks, то key -> valueпары:

'b' + 32-byte block hash -> block index record
'f' + 4-byte file number -> file information record
'l' -> 4-byte file number: the last block file number used
'R' -> 1-byte boolean: whether we're in the process of reindexing
'F' + 1-byte flag name length + flag name string -> 1 byte boolean: various flags that can be on or off
't' + 32-byte transaction hash -> transaction index record
```

В chainstate, то key -> valueпары:

'c' + 32-byte transaction hash -> unspent transaction output record for that transaction
'B' -> 32-byte block hash: the block hash up to which the database represents the unspent transaction outputs
(Подробное объяснение можно найти здесь )

Поскольку у нас еще нет транзакций, у нас будет только blocksкорзина. Также, как было сказано выше, мы будем хранить всю БД в виде одного файла, без сохранения блоков в отдельных файлах. Таким образом, нам не нужно ничего, что связано с номерами файлов. Итак, эти key -> valueпары мы будем использовать:

32-byte block-hash -> Block structure (serialized)
'l' -> the hash of the last block in a chain
Это все, что нам нужно знать, чтобы начать реализацию механизма персистентности.

## Сериализация
Как уже говорилось ранее, в BoltDB значения могут быть только []byteтипа, и мы хотим хранить Blockструктуры в БД. Мы будем использовать encoding / gob для сериализации структур.

Реализуем Serializeметод Block(обработка ошибок для краткости опущена):

```golang
func (b *Block) Serialize() []byte {
	var result bytes.Buffer
	encoder := gob.NewEncoder(&result)

	err := encoder.Encode(b)

	return result.Bytes()
}
```

Все просто: сначала мы объявляем буфер, который будет хранить сериализованные данные; затем мы инициализируем gob 
кодировщик и кодируем блок; результат возвращается как байтовый массив.


Далее нам нужна десериализационная функция, которая получит байтовый массив в качестве входных данных и вернет a Block. Это будет не метод, а независимая функция:

```golang

func DeserializeBlock(d []byte) *Block {
	var block Block

	decoder := gob.NewDecoder(bytes.NewReader(d))
	err := decoder.Decode(&block)

	return &block
}
```

И это все для сериализации!

## Упорство
Давайте начнем с NewBlockchainфункции. В настоящее время он создает новый экземпляр Blockchainи 
добавляет к нему блок genesis. Мы хотим, чтобы это было:

Откройте файл БД.  
Проверьте, есть ли в нем блокчейн.  

Если есть блокчейн:  
Создайте новый Blockchainэкземпляр.
Установите кончик Blockchainэкземпляра на последний хэш блока, сохраненный в БД.

Если не существует блокчейна:
Создайте блок генезиса.
Хранить в БД.
Сохраните хеш блока Genesis как последний хеш блока.
Создайте новый Blockchainэкземпляр с его подсказкой, указывающей на блок генеза.

В коде это выглядит так:

```golang
func NewBlockchain() *Blockchain {
	var tip []byte
	db, err := bolt.Open(dbFile, 0600, nil)

	err = db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))

		if b == nil {
			genesis := NewGenesisBlock()
			b, err := tx.CreateBucket([]byte(blocksBucket))
			err = b.Put(genesis.Hash, genesis.Serialize())
			err = b.Put([]byte("l"), genesis.Hash)
			tip = genesis.Hash
		} else {
			tip = b.Get([]byte("l"))
		}

		return nil
	})

	bc := Blockchain{tip, db}

	return &bc
}
```

Давайте рассмотрим этот кусок за куском.

```golang
db, err := bolt.Open(dbFile, 0600, nil)
```

Это стандартный способ открытия файла BoltDB. Обратите внимание, что он не вернет ошибку, если такого файла нет.

```golang
err = db.Update(func(tx *bolt.Tx) error {
...
})
```

В BoltDB операции с базой данных выполняются внутри транзакции.
И есть два типа транзакций: только чтение и чтение-запись. 
Здесь мы открываем транзакцию чтения-записи ( db.Update(...)), потому что мы ожидаем поместить блок генеза в БД.

```golang
b := tx.Bucket([]byte(blocksBucket))

if b == nil {
	genesis := NewGenesisBlock()
	b, err := tx.CreateBucket([]byte(blocksBucket))
	err = b.Put(genesis.Hash, genesis.Serialize())
	err = b.Put([]byte("l"), genesis.Hash)
	tip = genesis.Hash
} else {
	tip = b.Get([]byte("l"))
}
```

Это ядро ​​функции. Здесь мы получаем корзину, в которой хранятся наши блоки: если она существует,
мы читаем lключ из нее; если он не существует, мы генерируем блок genesis, создаем сегмент,
сохраняем в нем блок и обновляем lключ, хранящий хэш последнего блока цепочки.

Также обратите внимание на новый способ создания Blockchain:

```golang

bc := Blockchain{tip, db}
```

Мы больше не храним в нем все блоки, вместо этого хранится только верхушка цепочки. Кроме того, мы храним соединение с БД, потому что хотим открыть его один раз и оставить открытым во время работы программы. Таким образом, Blockchainструктура теперь выглядит так:

```golang

type Blockchain struct {
	tip []byte
	db  *bolt.DB
}
```

Следующее, что мы хотим обновить, - это AddBlockметод: добавить блоки в цепочку сейчас не так просто, как добавить элемент в массив. Отныне мы будем хранить блоки в БД:

```golang
func (bc *Blockchain) AddBlock(data string) {
	var lastHash []byte

	err := bc.db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		lastHash = b.Get([]byte("l"))

		return nil
	})

	newBlock := NewBlock(data, lastHash)

	err = bc.db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		err := b.Put(newBlock.Hash, newBlock.Serialize())
		err = b.Put([]byte("l"), newBlock.Hash)
		bc.tip = newBlock.Hash

		return nil
	})
}
```

Давайте рассмотрим этот кусок за куском:

```golang

err := bc.db.View(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte(blocksBucket))
	lastHash = b.Get([]byte("l"))

	return nil
})
```

Это другой (только для чтения) тип транзакций BoltDB. Здесь мы получаем последний хэш блока из БД, 
чтобы использовать его для извлечения нового хеша блока.

```golang
newBlock := NewBlock(data, lastHash)
b := tx.Bucket([]byte(blocksBucket))
err := b.Put(newBlock.Hash, newBlock.Serialize())
err = b.Put([]byte("l"), newBlock.Hash)
bc.tip = newBlock.Hash
```

После извлечения нового блока мы сохраняем его сериализованное представление в БД и обновляем lключ, 
который теперь хранит хэш нового блока.

Готово! Это было не сложно, не так ли?

## Проверка блокчейна
Все новые блоки теперь сохраняются в базе данных, поэтому мы можем открыть блокчейн и добавить в него новый блок. 
Но после реализации мы потеряли приятную особенность: мы больше не можем распечатывать блоки блокчейна, 
потому что мы больше не храним блоки в массиве. Давайте исправим этот недостаток!

***BoltDB*** позволяет перебирать все ключи в сегменте, но ключи хранятся в порядке сортировки байтов, и мы хотим, 
чтобы блоки печатались в порядке, который они принимают в блокчейне. Кроме того, поскольку мы не хотим загружать 
все блоки в память (наша БД может быть огромной! .. или просто притворимся, что это возможно), мы будем читать их 
один за другим. Для этого нам понадобится итератор блокчейна:

```golang

type BlockchainIterator struct {
	currentHash []byte
	db          *bolt.DB
}
```

Итератор будет создаваться каждый раз, когда мы захотим перебирать блоки в блокчейне, и он будет 
хранить хэш блока текущей итерации и соединение с БД. Из-за последнего итератор логически присоединяется
к блокчейну (это Blockchainэкземпляр, который хранит соединение с БД) и, т
аким образом, создается в Blockchain методе:

```golang

func (bc *Blockchain) Iterator() *BlockchainIterator {
	bci := &BlockchainIterator{bc.tip, bc.db}

	return bci
}
```

Обратите внимание, что итератор изначально указывает на верхушку блокчейна, поэтому блоки будут получены сверху вниз, 
от самого нового до самого старого. Фактически, выбор чаевых означает «голосование» за блокчейн . 
Блокчейн может иметь несколько ветвей, и он считается самым длинным из них основным. 
После получения подсказки (это может быть любой блок в блокчейне) мы можем реконструировать весь блокчейн и 
найти его длину и работу, необходимую для его построения. Этот факт также означает, что подсказка является 
своего рода идентификатором блокчейна.


## BlockchainIterator 
сделает только одно: он вернет следующий блок из блокчейна.

```golang
func (i *BlockchainIterator) Next() *Block {
	var block *Block

	err := i.db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		encodedBlock := b.Get(i.currentHash)
		block = DeserializeBlock(encodedBlock)

		return nil
	})

	i.currentHash = block.PrevBlockHash

	return block
}
```

Вот и все для части БД!

## CLI
До сих пор наша реализация не не предоставила интерфейс для взаимодействия с программой: 
мы просто выполняются NewBlockchain, bc.AddBlockв mainфункции. 
Время улучшить это! Мы хотим иметь эти команды:

```golang
blockchain_go addblock "Pay 0.031337 for a coffee"
blockchain_go printchain
```

Все связанные с командной строкой операции будут обрабатываться CLIструктурой:

```golang
type CLI struct {
	bc *Blockchain
}
```

Его «точкой входа» является Runфункция:

```golang
func (cli *CLI) Run() {
	cli.validateArgs()

	addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
	printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)

	addBlockData := addBlockCmd.String("data", "", "Block data")

	switch os.Args[1] {
	case "addblock":
		err := addBlockCmd.Parse(os.Args[2:])
	case "printchain":
		err := printChainCmd.Parse(os.Args[2:])
	default:
		cli.printUsage()
		os.Exit(1)
	}

	if addBlockCmd.Parsed() {
		if *addBlockData == "" {
			addBlockCmd.Usage()
			os.Exit(1)
		}
		cli.addBlock(*addBlockData)
	}

	if printChainCmd.Parsed() {
		cli.printChain()
	}
}
```

Мы используем стандартный пакет флагов для разбора аргументов командной строки.

```golang
addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)
addBlockData := addBlockCmd.String("data", "", "Block data")
```

Сначала мы создаем две подкоманды, addblockа printchainзатем добавляем -dataфлаг к первой. printchain 
не будет никаких флагов

```golang
switch os.Args[1] {
case "addblock":
	err := addBlockCmd.Parse(os.Args[2:])
case "printchain":
	err := printChainCmd.Parse(os.Args[2:])
default:
	cli.printUsage()
	os.Exit(1)
}
```

Далее мы проверяем команду, предоставленную пользователем, и анализируем связанную flagподкоманду.

```golang
if addBlockCmd.Parsed() {
	if *addBlockData == "" {
		addBlockCmd.Usage()
		os.Exit(1)
	}
	cli.addBlock(*addBlockData)
}

if printChainCmd.Parsed() {
	cli.printChain()
}
```

Далее мы проверяем, какие из подкоманд были проанализированы, и запускаем связанные функции.

```golang
func (cli *CLI) addBlock(data string) {
	cli.bc.AddBlock(data)
	fmt.Println("Success!")
}

func (cli *CLI) printChain() {
	bci := cli.bc.Iterator()

	for {
		block := bci.Next()

		fmt.Printf("Prev. hash: %x\n", block.PrevBlockHash)
		fmt.Printf("Data: %s\n", block.Data)
		fmt.Printf("Hash: %x\n", block.Hash)
		pow := NewProofOfWork(block)
		fmt.Printf("PoW: %s\n", strconv.FormatBool(pow.Validate()))
		fmt.Println()

		if len(block.PrevBlockHash) == 0 {
			break
		}
	}
}
```

Эта часть очень похожа на ту, что была у нас раньше. Единственное отличие состоит в том, 
что мы теперь используем BlockchainIteratorдля перебора блоков в блокчейне.

Также давайте не забудем изменить mainфункцию соответственно:

```golang

func main() {
	bc := NewBlockchain()
	defer bc.db.Close()

	cli := CLI{bc}
	cli.Run()
}
```

Обратите внимание, что новый Blockchainсоздается независимо от того, какие аргументы командной строки предоставляются.

И это все! Давайте проверим, что все работает как положено:

```
$ blockchain_go printchain
No existing blockchain found. Creating a new one...
Mining the block containing "Genesis Block"
000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true

$ blockchain_go addblock -data "Send 1 BTC to Ivan"
Mining the block containing "Send 1 BTC to Ivan"
000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13

Success!

$ blockchain_go addblock -data "Pay 0.31337 BTC for a coffee"
Mining the block containing "Pay 0.31337 BTC for a coffee"
000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148

Success!

$ blockchain_go printchain
Prev. hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
Data: Pay 0.31337 BTC for a coffee
Hash: 000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148
PoW: true

Prev. hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
Data: Send 1 BTC to Ivan
Hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
PoW: true

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true
```

(звук открывающейся банки с пивом)

## Заключение
В следующий раз мы реализуем адреса, кошельки и (возможно) транзакции. Так что следите за обновлениями!
