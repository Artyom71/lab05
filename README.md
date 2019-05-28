## Laboratory work V

Данная лабораторная работа посвещена изучению фреймворков для тестирования на примере **GTest**

```ShellSession
$ open https://github.com/google/googletest
```

## Tasks

- [ ] 1. Создать публичный репозиторий с названием **lab05** на сервисе **GitHub**
- [ ] 2. Выполнить инструкцию учебного материала
- [ ] 3. Ознакомиться со ссылками учебного материала
- [ ] 4. Составить отчет и отправить ссылку личным сообщением в **Slack**

## Tutorial

```ShellSession
$ export GITHUB_USERNAME=<имя_пользователя>
$ alias gsed=sed # for *-nix system
```

```ShellSession
$ cd ${GITHUB_USERNAME}/workspace
$ pushd .
$ source scripts/activate
```

```ShellSession
$ git clone https://github.com/${GITHUB_USERNAME}/lab04 projects/lab05
$ cd projects/lab05
$ git remote remove origin
$ git remote add origin https://github.com/${GITHUB_USERNAME}/lab05
```

```ShellSession
$ mkdir third-party
$ git submodule add https://github.com/google/googletest third-party/gtest
$ cd third-party/gtest && git checkout release-1.8.1 && cd ../..
$ git add third-party/gtest
$ git commit -m"added gtest framework"
```

```ShellSession
$ gsed -i '/option(BUILD_EXAMPLES "Build examples" OFF)/a\
option(BUILD_TESTS "Build tests" OFF)
' CMakeLists.txt
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

```ShellSession
$ mkdir tests
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

```ShellSession
$ cmake -H. -B_build -DBUILD_TESTS=ON
$ cmake --build _build
$ cmake --build _build --target test
```

```ShellSession
$ _build/check
$ cmake --build _build --target test -- ARGS=--verbose
```

```ShellSession
$ gsed -i 's/lab04/lab05/g' README.md
$ gsed -i 's/\(DCMAKE_INSTALL_PREFIX=_install\)/\1 -DBUILD_TESTS=ON/' .travis.yml
$ gsed -i '/cmake --build _build --target install/a\
- cmake --build _build --target test -- ARGS=--verbose
' .travis.yml
```

```ShellSession
$ travis lint
```

```ShellSession
$ git add .travis.yml
$ git add tests
$ git add -p
$ git commit -m"added tests"
$ git push origin master
```

```ShellSession
$ travis login --auto
$ travis enable
```

```ShellSession
$ mkdir artifacts
$ sleep 20s && gnome-screenshot --file artifacts/screenshot.png
# for macOS: $ screencapture -T 20 artifacts/screenshot.png
# open https://github.com/${GITHUB_USERNAME}/lab05
```

## Report

```ShellSession
$ popd
$ export LAB_NUMBER=05
$ git clone https://github.com/tp-labs/lab${LAB_NUMBER} tasks/lab${LAB_NUMBER}
$ mkdir reports/lab${LAB_NUMBER}
$ cp tasks/lab${LAB_NUMBER}/README.md reports/lab${LAB_NUMBER}/REPORT.md
$ cd reports/lab${LAB_NUMBER}
$ edit REPORT.md
$ gistup -m "lab${LAB_NUMBER}"
```

## Homework

### Задание
1. Создайте `CMakeList.txt` для библиотеки *banking*.
2. Создайте модульные тесты на классы `Transaction` и `Account`.
    * Используйте mock-объекты.
    * Покрытие кода должно составлять 100%.
3. Настройте сборочную процедуру на **TravisCI**.
4. Настройте [Coveralls.io](https://coveralls.io/).

### Выполнение задания

1. Редактируем /banking/CMakeLists.txt.
```cmake
cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

project(banking)

add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)
```

Указываем имя проекта, версию cmake, стандарт языка программирования.
Добавляем в библиотеку Transaction.cpp и Account.cpp файла, указываем папку, где хранятся эти файлы.
Исправим ошибки в /banking/CMakeLists.txt:
```cmake
cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TEST "Build tests" OFF)

project(banking)

add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)

if(BUILD_TESTS)
	enable_testing()
	add_subdirectory(submodules/gtest)
	add_executable(check tests/tests.cpp)
	target_link_libraries(check banking gtest_main gmock_main)
	add_test(NAME check COMMAND check)
endif()
```

Добавим секцию if. Cначала происходит тестирование текущего репозитория. Далее подключается поддиректория с библиотекой gtest и указываем: файл, который будет собираться, библиотеки для подключения к этому файлу. 
В add_test мы указываем имя собираемого файла после NAME и после COMMAND. Ничего не указваем, так как если указать имя собираемого файла, то в конечном итоге туда подставится его положение.


2. Модульные тесты.

tests/tests.cpp:
```cpp
#include "Account.h"
#include "Transaction.h"
#include <gtest/gtest.h>
#include <gmock/gmock.h>

class AccountMock : public Account {
public:
	AccountMock(int id, int balance) : Account(id, balance) {}
	MOCK_CONST_METHOD0(GetBalance, int());
	MOCK_METHOD1(ChangeBalance, void(int diff));
	MOCK_METHOD0(Lock, void());
	MOCK_METHOD0(Unlock, void());
};
class TransactionMock : public Transaction {
public:
	MOCK_METHOD3(Make, bool(Account& from, Account& to, int sum));
};

TEST(Account, Mock) {
	AccountMock acc(1, 100);
	EXPECT_CALL(acc, GetBalance()).Times(1);
	EXPECT_CALL(acc, ChangeBalance(testing::_)).Times(2);
	EXPECT_CALL(acc, Lock()).Times(2);
	EXPECT_CALL(acc, Unlock()).Times(1);
	acc.GetBalance();
	acc.ChangeBalance(100); // throw
	acc.Lock();
	acc.ChangeBalance(100);
	acc.Lock(); // throw
	acc.Unlock();
}

TEST(Account, SimpleTest) {
	Account acc(1, 100);
	EXPECT_EQ(acc.id(), 1);
	EXPECT_EQ(acc.GetBalance(), 100);
	EXPECT_THROW(acc.ChangeBalance(100), std::runtime_error);
	EXPECT_NO_THROW(acc.Lock());
	acc.ChangeBalance(100);
	EXPECT_EQ(acc.GetBalance(), 200);
	EXPECT_THROW(acc.Lock(), std::runtime_error);
}

TEST(Transaction, Mock) {
	TransactionMock tr;
	Account ac1(1, 50);
	Account ac2(2, 500);
	EXPECT_CALL(tr, Make(testing::_, testing::_, testing::_))
	.Times(6);
	tr.set_fee(100);
	tr.Make(ac1, ac2, 199);
	tr.Make(ac2, ac1, 500);
	tr.Make(ac2, ac1, 300);
	tr.Make(ac1, ac1, 0);
	tr.Make(ac1, ac2, -1);
	tr.Make(ac1, ac2, 99);
}

TEST(Transaction, SimpleTest) {
	Transaction tr;
	Account ac1(1, 50);
	Account ac2(2, 500);
	tr.set_fee(100);
	EXPECT_EQ(tr.fee(), 100);
	EXPECT_THROW(tr.Make(ac1, ac1, 0), std::logic_error);
	EXPECT_THROW(tr.Make(ac1, ac2, -1), std::invalid_argument);
	EXPECT_THROW(tr.Make(ac1, ac2, 99), std::logic_error);
	EXPECT_FALSE(tr.Make(ac1, ac2, 199));
	EXPECT_FALSE(tr.Make(ac2, ac1, 500));
	EXPECT_TRUE(tr.Make(ac2, ac1, 300));
}
```

 Важно заметить, что наследование происходит с меткой *public*.
 Далее, с помощью EXPECT_CALL и .Times() мы задаем сколько раз будет вызываться данная функция. Если вызвана меньше - тест не пройдет.

3. Настроим Travis-Ci

Создаем .travis.yml

```yaml
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
```

4. Coveralls.io

Добавим файл .coveralls.yml в корень репозитория:

```yaml
service_name: travis-pro
repo_token: e0d421b980b19bf082c17ffe9fb75923c9d40ff5
```

 Исправим CMakeLists.txt:

```cmake
cmake_minimum_required(VERSION 3.4)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(COVERAGE "Check coverage" ON)

project(banking)

add_library(banking STATIC banking/Account.cpp banking/Transaction.cpp)
target_include_directories(banking PUBLIC banking/)

if (COVERAGE)
	target_compile_options(check PRIVATE --coverage)
	target_link_libraries(check --coverage)
endif()
``` 

Это нужно, чтобы при компиляции создавались определенные файлы, нужные для coveralls.
Исправим .travis.yml:

```yaml
before_install:
- pip install --user cpp-coveralls
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
after_success:
- coveralls --root . -E ".*gtest.*" -E ".*CMakeFiles.*"
```

## Links

- [C++ CI: Travis, CMake, GTest, Coveralls & Appveyor](http://david-grs.github.io/cpp-clang-travis-cmake-gtest-coveralls-appveyor/)
- [Boost.Tests](http://www.boost.org/doc/libs/1_63_0/libs/test/doc/html/)
- [Catch](https://github.com/catchorg/Catch2)

```
Copyright (c) 2015-2019 The ISC Authors
```
