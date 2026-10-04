## Data Connectorについて詳しく教えて下さい。また、Data attributesとの違いは?

はい。Data attributes と Data Connectors は、どちらも顧客情報を扱いますが、役割が異なります。
Data attributes：Intercom内に保存するユーザー／会社の項目です。
例：Account name、Company ID、契約プラン、担当者、利用状況。
一度Intercomに取り込めば、セグメント、ワークフロー、レポート、メッセージの条件などで利用できます。
Data Connectors：Intercomから、必要なタイミングで外部システムのREST APIを呼び出して情報を取得・更新する仕組みです。
例：問い合わせ中に、既存システムのAPIから「最新の契約状況」「注文状況」「障害情報」を取得し、Finやワークフローで参照する、といった使い方です。