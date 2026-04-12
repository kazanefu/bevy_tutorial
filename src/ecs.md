# ECSとは

ECSはEntity Component Systemの略で、UnityではDOTS(Data-Oriented Technology Stack)とも呼ばれることがあります。

- Entity: オブジェクト的なもののID
- Component: 要素、属性、データ
- System: 処理、ロジック

EntityはUnityのOOPでいうGameObjectのようなものです。Entity自体は何も持っていません。IDのようなものです。
ComponentはUnityのOOPでいうComponentとあまり違いはありませんが、OOPのComponentは動作も持ちますが、ECSのComponentは構造体でデータのみを持ちます。また、付け替えが容易です。
SystemはUnityでいうScript内のメソッドのようなものです。Systemは処理を持ち、Componentを操作します。

ECSの利点は、データがメモリ上に連続して配置されるため、キャッシュ効率が良く、高速な処理が可能です。また、データと処理が分離されているため、再利用性が高く、保守性の高いコードを書くことができます。
