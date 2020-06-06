## Laboratory work V

<a href="https://yandex.ru/efir/?stream_id=vQw_LH0UfN6I"><img src="https://raw.githubusercontent.com/tp-labs/lab05/master/preview.png" width="640"/></a>

Данная лабораторная работа посвещена изучению фреймворков для тестирования на примере **GTest**

```sh
$ open https://github.com/google/googletest
```

## Tasks

- [ ] 1. Создать публичный репозиторий с названием **lab05** на сервисе **GitHub**
- [ ] 2. Выполнить инструкцию учебного материала
- [ ] 3. Ознакомиться со ссылками учебного материала
- [ ] 4. Составить отчет и отправить ссылку личным сообщением в **Slack**

## Tutorial

```sh
$ export GITHUB_USERNAME=<имя_пользователя>
$ alias gsed=sed # for *-nix system
```

```sh
$ cd ${GITHUB_USERNAME}/workspace
$ pushd .
$ source scripts/activate
```

```sh
# Клонируем репозиторий с прошлой лабой
$ git clone https://github.com/${GITHUB_USERNAME}/lab04 projects/lab05
$ cd projects/lab05
$ git remote remove origin
$ git remote add origin https://github.com/${GITHUB_USERNAME}/lab05
```

```sh
$ mkdir third-party
$ git submodule add https://github.com/google/googletest third-party/gtest # Подключаем к епозиторию подмодуль GoogleTest
$ cd third-party/gtest && git checkout release-1.8.1 && cd ../.. # Настраиваем версию
$ git add third-party/gtest # коммитим
$ git commit -m"added gtest framework"
```


```sh
# добавляем опцию для сборки тестов
$ gsed -i '/option(BUILD_EXAMPLES "Build examples" OFF)/a\ 
option(BUILD_TESTS "Build tests" OFF)
' CMakeLists.txt
# вставка в конец файла
$ cat >> CMakeLists.txt <<EOF

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(third-party/gtest)
  file(GLOB \${PROJECT_NAME}_TEST_SOURCES tests/*.cpp)
  add_executable(check \${\${PROJECT_NAME}_TEST_SOURCES})
  target_link_libraries(check \${PROJECT_NAME} gtest_main)
  add_test(NAME check COMMAND check)
endif()
EOF
```

```sh
$ mkdir tests
# пишем первый тест
$ cat > tests/test1.cpp <<EOF
#include <print.hpp>

#include <gtest/gtest.h>

TEST(Print, InFileStream)
{
  std::string filepath = "file.txt";
  std::string text = "hello";
  std::ofstream out{filepath};

  print(text, out);
  out.close();

  std::string result;
  std::ifstream in{filepath};
  in >> result;

  EXPECT_EQ(result, text);
}
EOF
```

```sh
# генерация файлов для сборки с тестом
$ cmake -H. -B_build -DBUILD_TESTS=ON
$ cmake --build _build # сборка проекта
$ cmake --build _build --target test # сборка проекта и теста
```

```sh
$ _build/check # запуск теста
$ cmake --build _build --target test -- ARGS=--verbose # запуск с полным выводом
```

```sh
# модифицируем .travis.yml
$ gsed -i 's/lab04/lab05/g' README.md 
$ gsed -i 's/\(DCMAKE_INSTALL_PREFIX=_install\)/\1 -DBUILD_TESTS=ON/' .travis.yml 
$ gsed -i '/cmake --build _build --target install/a\
- cmake --build _build --target test -- ARGS=--verbose 
' .travis.yml
```

```sh
$ travis lint # проверяем travis.yml на наличие ошибок
```

```sh
# заливаем в удаленный репозиторий
$ git add .travis.yml
$ git add tests
$ git add -p
$ git commit -m"added tests"
$ git push origin master
```

```sh
# включаем travis
$ travis login --auto
$ travis enable
```

```sh
# делаем скриншот
$ mkdir artifacts
$ sleep 20s && gnome-screenshot --file artifacts/screenshot.png
# for macOS: $ screencapture -T 20 artifacts/screenshot.png
# open https://github.com/${GITHUB_USERNAME}/lab05
```

## Report

```sh
$ popd
$ export LAB_NUMBER=05
$ git clone https://github.com/tp-labs/lab${LAB_NUMBER} tasks/lab${LAB_NUMBER}
$ mkdir reports/lab${LAB_NUMBER}
$ cp tasks/lab${LAB_NUMBER}/README.md reports/lab${LAB_NUMBER}/REPORT.md
$ cd reports/lab${LAB_NUMBER}
$ edit REPORT.md
$ gist REPORT.md
```

## Homework

### Задание
1. Создайте `CMakeList.txt` для библиотеки *banking*.
2. Создайте модульные тесты на классы `Transaction` и `Account`.
    * Используйте mock-объекты.
    * Покрытие кода должно составлять 100%.
3. Настройте сборочную процедуру на **TravisCI**.
4. Настройте [Coveralls.io](https://coveralls.io/).


## Report
1. Создадим CMakeList.txt в директории banking
```sh
$ cat > banking/CMakeLists.txt <<EOF
project(banking_lib)

if (NOT TARGET libbanking)
    add_library(libbanking STATIC
        ${CMAKE_CURRENT_LIST_DIR}/Account.cpp
        ${CMAKE_CURRENT_LIST_DIR}/Transaction.cpp
    )

    install(TARGETS libbanking
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
    )
endif(NOT TARGET libbanking)

include_directories(${CMAKE_CURRENT_LIST_DIR})
EOF
```

2. Напишем тесты отдельно для класса Transaction:

```cpp
#include "Account.h"
#include "Transaction.h"

#include <gtest/gtest.h>
#include <gmock/gmock.h>

using testing::_;

class TransactionMock : public Transaction {
public:
    MOCK_METHOD3(Make, bool(Account& from, Account& to, int sum));
};

TEST(Transaction, Mock) {
    TransactionMock tr;
    Account acc1(1, 100);
    Account acc2(2, 300);

    EXPECT_CALL(tr, Make(_, _, _)).Times(5);
    tr.set_fee(200);
    tr.Make(acc1, acc2, 200);
    tr.Make(acc2, acc1, 300);
    // throws
    tr.Make(acc2, acc1, 50);
    tr.Make(acc2, acc1, 0);
    tr.Make(acc1, acc2, -8);
}

TEST(Transaction, TestTransaction) {
    Transaction tr;
    Account a1(1, 100), a2(2, 300);
    tr.set_fee(10);
    EXPECT_EQ(tr.fee(), 10);
    EXPECT_THROW(tr.Make(a1, a2, 90), std::logic_error);
    EXPECT_THROW(tr.Make(a1, a2, -1), std::invalid_argument);
    EXPECT_THROW(tr.Make(a1, a1, 100), std::logic_error);
    EXPECT_FALSE(tr.Make(a1, a2, 400));
    EXPECT_FALSE(tr.Make(a2, a1, 300));
    EXPECT_TRUE(tr.Make(a2, a1, 246));
}
```

И отдельно для Account:

```cpp

```#include "Account.h"
#include "Transaction.h"

#include <gtest/gtest.h>
#include <gmock/gmock.h>

using testing::_;

class AccountMock : public Account {
public:
    explicit AccountMock(int id, int balance) : Account(id, balance) {}
    MOCK_CONST_METHOD0(GetBalance, int());
    MOCK_METHOD1(ChangeBalance, void(int diff));
    MOCK_METHOD0(Lock, void());
    MOCK_METHOD0(Unlock, void());
};

TEST(Account, Mock) {
    AccountMock acc(1, 666);
    EXPECT_CALL(acc, GetBalance()).Times(1);
    EXPECT_CALL(acc, ChangeBalance(_)).Times(2);
    EXPECT_CALL(acc, Lock()).Times(2);
    EXPECT_CALL(acc, Unlock()).Times(1);
    acc.GetBalance();
    acc.ChangeBalance(100);
    acc.Lock();
    acc.ChangeBalance(100);
    acc.Lock();
    acc.Unlock();
}

TEST(Account, TestAccount) {
    Account acc(1, 666);
    EXPECT_EQ(acc.id(), 1);
    EXPECT_EQ(acc.GetBalance(), 666);
    EXPECT_THROW(acc.ChangeBalance(200), std::runtime_error);
    EXPECT_NO_THROW(acc.Lock());
    acc.ChangeBalance(200);
    EXPECT_EQ(acc.GetBalance(), 866);
    EXPECT_THROW(acc.Lock(), std::runtime_error);
    EXPECT_NO_THROW(acc.Unlock());
}
```


CMakeLists.txt для сборки с тестами:
```sh
project(banking)

cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDART 11)
set(CMAKE_CXX_STANDART_REQUIRED ON)

option(BUILD_TESTS "Build tests" OFF)

option(COVERAGE ON)

include(${CMAKE_CURRENT_SOURCE_DIR}/banking/CMakeLists.txt)

if (BUILD_TESTS)
    enable_testing()
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/third-party/gtest)
    if (TARGET libbanking)
        # test Account module
        add_executable(check_account ${CMAKE_CURRENT_SOURCE_DIR}/banking/tests/test_account.cpp)
        target_link_libraries(check_account libbanking gtest_main gmock_main)
        add_test(NAME account COMMAND check_account)

        # test Transaction module
        add_executable(check_transaction ${CMAKE_CURRENT_SOURCE_DIR}/banking/tests/test_transaction.cpp)
        target_link_libraries(check_transaction libbanking gtest_main gmock_main)
        add_test(NAME transaction COMMAND check_transaction)

        if (COVERAGE)
            target_compile_options(check_account PRIVATE --coverage)
            target_compile_options(check_transaction PRIVATE --coverage)

            target_link_libraries(check_account --coverage)
            target_link_libraries(check_transaction --coverage)
        endif(COVERAGE)
    endif(TARGET libbanking)
endif(BUILD_TESTS)

before_install:
- pip install --user cpp-coveralls

after_success:
- coveralls --root . -E ".*gtest.*" -E ".*CMakeFiles.*"
```

файл travis.yml:
```sh
language: cpp
os:
  - linux
addons:
  apt:
    sources:
    - george-edison55-precise-backports
    packages:
    - cmake
    - cmake-data
script:
- cmake -H. -B_build -DBUILD_TESTS=ON
- cmake --build _build
- cmake --build _build --target test -- ARGS=--verbose
before_install:
- pip install --user cpp-coveralls
after_success:
- coveralls --root . -E ".*gtest.*" -E ".*CMakeFiles.*"
```

файл coveralls.yml:
```
service_name: travis-pro
repo_token: <мой_токен>
```
## Links

- [C++ CI: Travis, CMake, GTest, Coveralls & Appveyor](http://david-grs.github.io/cpp-clang-travis-cmake-gtest-coveralls-appveyor/)
- [Boost.Tests](http://www.boost.org/doc/libs/1_63_0/libs/test/doc/html/)
- [Catch](https://github.com/catchorg/Catch2)

```
Copyright (c) 2015-2020 The ISC Authors
```
