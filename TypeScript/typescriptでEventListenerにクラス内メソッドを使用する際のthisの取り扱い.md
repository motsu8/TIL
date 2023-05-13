## 結論
`Function.bind()`で参照したいオブジェクトをバインドさせるとスマート。

## 背景
クラスメソッドでEventTargetに追加したListenerを削除する必要ができたため、無名関数として入れていたものをオブジェクトとして渡す際に期待した挙動をしなかった。

### 元の状態
```typescript
class Temp{
    private state: boolean
    constructor(){
        this.state = true;
    }

    public start(): void{
        ...
        document.addEventListener('keydown', (e)=>{
            ...
            switch(e.code){
                case...:
                    this.move();
                    break;
            }
        });
    };
    ...
}
```

## Listenerを削除するため、無名関数を関数へ変更
EventTargetに追加されるEventListenerは関数の参照が渡されるため、addEventListener内で無名関数を定義すると参照が保持されない。
そのため、EventTargetに追加された無名関数は後々削除することができない。
~~Listenerを削除する必要がある場合は、~~ 基本的に、addEventListener内で無名関数を記述しないほうが良さそう。
```typescript
    public start(): void{
        ...
        document.addEventListener('keydown', this.moveCtrl);
    }

    public moveCtrl(e: KeyBoardEvent): void{
            ...
            switch(e.code){
                case...:
                    this.move();
                    break;
            }
    };
```

### this問題
ここでthisはどこを参照してくるか？
```typescript
    // 実行時
    this.moveCtrl => Temp.moveCtrl
    this.state    => document.state() // undefined
    this.move     => document.move() // undefined

    // 理想
    this.moveCtrl => Temp.moveCtrl
    this.state    => Temp.state()
    this.move     => Temp.move()
```

## moveCtrl()内のthisをTempにさせる
thisをTempにさせるには以下の2つ
1. リスナーにオブジェクトとして渡す。
2. 渡す関数にTempをバインドする。

### 1. リスナーにオブジェクトとして渡す。
リスナーにオブジェクトを渡した場合、あらゆるイベントを捕捉する `handleEvent()`を指定することで実現できる。
ただし、イベントタイプで実行するメソッドを分けたい場合、`handleEvent()`内で条件分岐させる必要がありそう。
```typescript
    public start(): void{
        ...
        document.addEventListener('keydown', this);
    }

    // public moveCtrl(): void{
    public handleEvent(): void{
            ...
            if(this.state){
                switch(e.code){
                    case...:
                        this.move();
                        break;
                }
            }else{...}
    };
```

### 2. 渡す関数にTempをバインドする。
`Function.prototype.bind()`で関数内のthisの参照を指定することができる。
```typescript
    constructor(){
        this.moveCtrl = this.moveCtrl.bind(this)
    }

    public start(): void{
        ...
        document.addEventListener('keydown', this.moveCtrl);
    }

    public moveCtrl(e: KeyBoardEvent): void{
        ...
        switch(e.code){
            case...:
                this.move();
                break;
        }
    };
```
### 番外
いろいろこねくり回した結果
```typescript
type HandleEvent = {
    handleEvent: Function,
    manager: GameManager,
}

class Temp{
    private state: boolean
    private handleEvent: HandleEvent | EventListenerOrEventListenerObject;

    constructor(){
        this.state = true;
        this.handleEvent = {
            handleEvent: this.moveCtrl,
            manager: this,
        }
    }

    public start(): void{
        document.addEventListener("keydown", (this.handleEvent as EventListenerOrEventListenerObject));
    }

    public moveCtrl(e: KeyBoardEvent): void{
        ...
        switch(e.code){
            case...:
                (this as any as HandleEvent).manager.move();
                break;
        }
    };
}
```

## 参考文献
https://developer.mozilla.org/ja/docs/Web/API/EventTarget/addEventListener#%E3%81%9D%E3%81%AE%E4%BB%96%E3%81%AE%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A0%85
https://note.com/yamanoborer/n/n2e4cc40328b7