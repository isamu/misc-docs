 
- lib
  - SlashGPTのプログラム一式
- manifests
  - マニフェストファイル一式
- config
  - llm/llm engineの設定ファイル
- resources
  - templates
    - resouceパラメータで読み込まれ、システムプロンプトの{resource}部分に置換される
  - funcitons
    - llmにfunctions callのパラメータとして渡される
  - module
    - moduleで指定されるPythonのファイルで、function callの戻り値により実行される
  - action_template
    - actionsの中でtemplateと指定されるファイルで、function callの戻り値により実行されるactionから読み込まれるファイル
- data
  - functionsから利用するデータ
- notebooks
  - notebooks = true時に読み込まれるipynb
- plugins
  - pluginで利用するLLM engine
- output
  - ログなどの出力先
- samples
  - SimpleGPT.pyやAgentGPT.pyなどのlib以下のmoduleを使ったサンプルプログラム
- test
  - autotestの設定ファイル
- tests
  - unit test
