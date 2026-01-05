## 個人情報

- 「社員情報」セクションに表示される項目は、API `/personal-profile/{employeeId}/employee-information` から取得されます。フォーマットは以下の通りです。

```json5
{
  data: [
    {
      key: "personal-information",
      label: "個人情報",
      progress: 55,
      lastUpdatedAt: "2019-12-01T09:21:33+00:00:00",
      items: [
        {
          key: "basic",
          label: "基本情報",
        },
        {
          key: "family",
          label: "家族情報",
        },
        // ...
      ],
    },
    // ...
  ],
}
```

- これは、ユーザーが閲覧権限を持つパネルと、各パネル内のカテゴリリストです。
- コントローラ `ApiPersonalProfileEmployeeInformationPanel` は、対応するRepositoryクラスを呼び出すことで、フロントエンドにこのデータを返す処理を担当します。

```php
<?php
// ...
$service = new PanelListGetApplicationService(
            new UserRepository(),
            new EmployeeRepository(),
            new EmployeeInformationItemRepository()
        );
```

- 特定のカテゴリ（例：categoryId=1000 (基本情報)）の詳細にアクセスする際、フロントエンドは `API /api/v1/personal-profile/{employeeId}/employee-information/1000` を呼び出し、このカテゴリの詳細データを取得します。

API `/api/v1/personal-profile/{employeeId}/employee-information/1000` を呼び出す際、主に以下のテーブルがクエリされます：

1.  `pp_personal_profile_category_groups` — パネルグループ

  目的：カテゴリをパネルにグループ化します（例：「基本情報」、「雇用情報」...）。
  重要なフィールド：
  • id: パネルID
  • key: パネルキー（例：personal-information, employment, ...）
  • is_use: 使用有無
  • display_order: 表示順

  ```text
  +----------------------------------+--------------+
  | Field                            | Type         |
  +----------------------------------+--------------+
  | id                               | int(11)      |
  | key                              | varchar(255) |
  | is_use                           | tinyint(1)   |
  | display_order                    | int(11)      |
  | excludes_in_employee_information | varchar(255) |
  | modified_by                      | varchar(255) |
  | created                          | datetime     |
  | modified                         | datetime     |
  +----------------------------------+--------------+
  ```

  例：
     id=1, key='personal-information', is_use=1, display_order=1
     id=2, key='employment', is_use=1, display_order=2

  ---

  2.  `pp_personal_profile_categories` — カテゴリ

  目的：パネル内のカテゴリを定義し、実際のデータテーブルにマッピングします。
  重要なフィールド：
  • id: カテゴリID（例：1000）
  • pp_personal_profile_category_group_id: FK → pp_personal_profile_category_groups.id
  • key: カテゴリキー（例：basic, contact）
  • table_name: データを格納するテーブル名（例：pp_employee_basics, pp_employee_employments）
  • record_type: レコードタイプ（single または multiple）
  • is_use: 使用有無
  • options: JSON設定（例：ソート順、フィルター）

  ```text
  +---------------------------------------+--------------+
  | Field                                 | Type         |
  +---------------------------------------+--------------+
  | id                                    | int(11)      |
  | pp_personal_profile_category_group_id | int(11)      |
  | key                                   | varchar(255) |
  | table_name                            | varchar(255) |
  | record_type                           | varchar(255) |
  | is_use                                | tinyint(1)   |
  | display_order                         | int(11)      |
  | is_snapshot                           | tinyint(1)   |
  | options                               | json         |
  | description                           | text         |
  | modified_by                           | varchar(255) |
  | created                               | datetime     |
  | modified                              | datetime     |
  +---------------------------------------+--------------+
  ```

  カテゴリID=1000の例：
     id=1000
     pp_personal_profile_category_group_id=1 (personal-information パネル)
     key='basic'
     table_name='pp_employee_basics'
     record_type='single'
     is_use=1

  → カテゴリ1000は「personal-information」パネルに属し、データは `pp_employee_basics` テーブルにあり、singleタイプ（従業員あたり1レコード）です。
  ---

  3.  `pp_personal_profile_items` — 項目/フィールド

  目的：カテゴリ内のフィールドを定義し、データテーブルの列にマッピングします。
  重要なフィールド：
  • id: 項目ID
  • pp_personal_profile_category_id: FK → pp_personal_profile_categories.id
  • key: 項目キー（例：last_name, first_name）
  • table_name: データを格納するテーブル名（通常はカテゴリと同じ。異なる場合もあり）
  • column_name: DB内の列名（例：last_name, first_name）
  • data_type: データタイプ（text, number, date, select, file, employee-search, none）
  • is_required_for_tenant: 必須かどうか
  • display_order: 表示順
  • options: JSON設定（例：select用のcode_type、ネスト用のparentId）

  ```text
  +---------------------------------+----------------+
  | Field                           | Type           |
  +---------------------------------+----------------+
  | id                              | int(11)        |
  | pp_personal_profile_category_id | int(11)        |
  | key                             | varchar(255)   |
  | table_name                      | varchar(255)   |
  | column_name                     | varchar(255)   |
  | data_type                       | varchar(255)   |
  | is_required_for_system          | tinyint(1)     |
  | is_required_for_tenant          | tinyint(1)     |
  | is_use                          | tinyint(1)     |
  | is_readonly                     | tinyint(1)     |
  | display_order                   | int(11)        |
  | digits                          | varchar(255)   |
  | integer_digits                  | int(11)        |
  | decimal_digits                  | int(11)        |
  | min                             | decimal(20,10) |
  | max                             | decimal(20,10) |
  | min_default                     | decimal(20,10) |
  | max_default                     | decimal(20,10) |
  | options                         | json           |
  | remarks                         | varchar(255)   |
  | modified_by                     | varchar(255)   |
  | created                         | datetime       |
  | modified                        | datetime       |
  +---------------------------------+----------------+
  ```

  カテゴリID=1000の例：

    id=1001, pp_personal_profile_category_id=1000, key='last_name',
            table_name='pp_employee_basics', column_name='last_name',
            data_type='text', display_order=1, is_required_for_system=1, is_required_for_tenant=1
    id=1002, pp_personal_profile_category_id=1000, key='middle_name',
            table_name='pp_employee_basics', column_name='middle_name',
            data_type='text', display_order=2, is_required_for_system=0, is_required_for_tenant=0
    id=1003, pp_personal_profile_category_id=1000, key='first_name',
            table_name='pp_employee_basics', column_name='first_name',
            data_type='text', display_order=3, is_required_for_system=1, is_required_for_tenant=1
    id=1004, pp_personal_profile_category_id=1000, key='phonetic_last_name',
            table_name='pp_employee_basics', column_name='phonetic_last_name',
            data_type='text', display_order=4, is_required_for_system=0, is_required_for_tenant=1
    id=1005, pp_personal_profile_category_id=1000, key='phonetic_middle_name',
            table_name='pp_employee_basics', column_name='phonetic_middle_name',
            data_type='text', display_order=5, is_required_for_system=0, is_required_for_tenant=0
    id=1006, pp_personal_profile_category_id=1000, key='phonetic_first_name',
            table_name='pp_employee_basics', column_name='phonetic_first_name',
            data_type='text', display_order=6, is_required_for_system=0, is_required_for_tenant=1
    id=1007, pp_personal_profile_category_id=1000, key='first_name_english',
            table_name='pp_employee_basics', column_name='first_name_english',
            data_type='text', display_order=7, is_required_for_system=0, is_required_for_tenant=0
    id=1008, pp_personal_profile_category_id=1000, key='middle_name_english',
            table_name='pp_employee_basics', column_name='middle_name_english',
            data_type='text', display_order=8, is_required_for_system=0, is_required_for_tenant=0
    id=1009, pp_personal_profile_category_id=1000, key='last_name_english',
            table_name='pp_employee_basics', column_name='last_name_english',
            data_type='text', display_order=9, is_required_for_system=0, is_required_for_tenant=0
    id=1010, pp_personal_profile_category_id=1000, key='sex',
            table_name='pp_employee_basics', column_name='gender_code',
            data_type='select', display_order=10, is_required_for_system=0, is_required_for_tenant=0
            (pp_personal_profile_item_code_types テーブルに pp_personal_profile_item_id=1010 のレコードがあり、
             codes テーブルと JOIN して、code_type と code_key に基づきラベルを取得します)
    id=1011, pp_personal_profile_category_id=1000, key='date_of_birth',
            table_name='pp_employee_basics', column_name='date_of_birth',
            data_type='date', display_order=11, is_required_for_system=0, is_required_for_tenant=0
    id=1012, pp_personal_profile_category_id=1000, key='age',
            table_name='pp_employee_basics', column_name='age',
            data_type='none', display_order=12, is_required_for_system=0, is_required_for_tenant=0
            (直接クエリせず、コード内で date_of_birth から計算します)
    id=1013, pp_personal_profile_category_id=1000, key='memo',
            table_name='pp_employee_basics', column_name='memo',
            data_type='textarea', display_order=13, is_required_for_system=0, is_required_for_tenant=0
    id=1014, pp_personal_profile_category_id=1000, key='business_name',
            table_name='pp_employee_basics', column_name='business_name',
            data_type='text', display_order=14, is_required_for_system=0, is_required_for_tenant=0
    id=1015, pp_personal_profile_category_id=1000, key='entered_in_common_kanji',
            table_name='pp_employee_basics', column_name='entered_in_common_kanji',
            data_type='select', display_order=15, is_required_for_system=0, is_required_for_tenant=0
            (pp_personal_profile_item_code_types テーブルに pp_personal_profile_item_id=1015 のレコードがあり、
             codes テーブルと JOIN して、code_type と code_key に基づきラベルを取得します)
    id=1016, pp_personal_profile_category_id=1000, key='date_of_death',
            table_name='pp_employee_basics', column_name='date_of_death',
            data_type='date', display_order=16, is_required_for_system=0, is_required_for_tenant=0

  ---

  3つのテーブルの関係性

     pp_personal_profile_category_groups (パネル)
         │
         ├─ 1:N ──→ pp_personal_profile_categories (カテゴリ)
                         │
                         ├─ 1:N ──→ pp_personal_profile_items (項目)

  ---
