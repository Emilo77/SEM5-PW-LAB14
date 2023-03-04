# Laboratorium - model pamięci w c++

### Dołączone pliki
Do laboratorium zostały dołączone następujące pliki:
<details><summary>:pushpin: Rozwiń </summary>
<p>

- [atomic-volatile.cpp](atomic-volatile.cpp)
- [CMakeLists.txt](CMakeLists.txt)
- [lockfree-stack.cpp](lockfree-stack.cpp)
- [mutual-excl.cpp](mutual-excl.cpp)

</p>
</details>

---

Dwie operacje pamięci są w *konflikcie*, gdy jedna zapisuje, a druga odczytuje ten sam adres. Wyścig (ang. *data race*) zachodzi, gdy te operacje wykonywane są równocześnie, a w modelu ze spójnością sekwencyjną (ang. *sequential consistency*) byłyby wykonywane jedna po drugiej. Języki programowania gwarantują spójność sekwencyjną wtedy i tylko wtedy, gdy kod nie ma wyścigów.

Na laboratorium będziemy eksplorować obrzeża modelu pamięci **C++14** implementując algorytmy wzajemnego wykluczania. W tym celu będziemy uruchamiać procesy w różnych wariantach:

- taskset: uruchamia proces na określonych procesorach, np: `taskset -c 0 ./mutual-excl` (procesor 0), albo `taskset -c 0,1 ./mutual-excl` (procesory 0 i 1)
- optymalizacje kompilatora (plik `CMakeLists.txt`, zmienna `CMAKE_CXX_FLAGS`): `O0` wyłącza optymalizację; kolejne liczby to coraz bardziej zaawansowane optymalizacje. Więcej informacji można znaleźć [tutaj]([https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html).
- brak wzajemnego wykluczania ([mutual-excl.cpp](mutual-excl.cpp)) vs. twoja implementacja algorytmu Petersona.

#### Zadanie 
Dla każdej kombinacji wariantów spróbuj przewidzieć poprawność wyniku.

### Implementacja algorytmu Petersona

Rozwiń szablon [mutual-excl.cpp](mutual-excl.cpp) implementując zawartość `entry_protocol` i `exit_protocol`.

## Zmienne volatile

Oznacz zmienne synchronizacyjne w algorytmie Petersona jako `volatile`.

`volatile` w **C++14** nie ma semantyki `volatile` jak w języku **java**. `volatile` powinno być używane tylko do oznaczenia zmiennych przechowywanych w specjalnej pamięci, tzn. pamięci, która może być zmieniana poza wykonywanym procesem (taką pamięcią jest np. `memory-mapped I/O`).

## Zmienne std::atomic

**C++14** pozwala na oznaczenie zmiennych jako [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic).  `std::atomic<T>` jest szablonem zdefiniowanym dla trywialnie kopiowalnego typu `T` (wartość takiego typu możemy ustawić poprzez zwykłe kopiowanie poszczególnych bajtów) udostępniającym następujące atomowe operacje (pełna lista: [http://en.cppreference.com/w/cpp/atomic/atomic](http://en.cppreference.com/w/cpp/atomic/atomic)):
```cpp
T load(); //zwraca aktualną wartość (równoważne z operatorem T())

void store(T desired); //zamienia wartość na desired (równoważne z operatorem =)

T exchange(T desired); //zamienia wartość na desired i zwraca poprzednią wartość

bool compare_exchange_weak(T& expected, T desired); 
// lub
bool compare_exchange_strong(T& expected, T desired); // o tym dalej w opisie
```

Oprócz tych operacji biblioteka standardowa oferuje specjalizacje `std::atomic` dla typów całkowitoliczbowych udostępniające atomowe funkcje m.in.:

- `fetch_add` / `fetch_sub`: zwróć dotychczasową wartość, następnie atomowo dodaj / odejmij argument
- `operator++` oraz `operator+=`.

Dla typów wskaźnikowych `std::atomic<T*>` dostępne są atomowe operacje:

- `fetch_add`: zwróć aktualną wartość wskaźnika, następnie przesuń go do przodu o argument;
- `fetch_sub`: zwróć aktualną wartość wskaźnika, następnie przesuń go do tyłu o argument.

Dodatkowo model pamięci C++14 traktuje operacje na zmienych `std::atomic` jak instrukcje synchronizacyjne, a nie jak zwykłe instrukcje odczytu/zapisu. Standardową semantyką jest spójność liniowa (`std::memory_order_seq_cst`), ale można wymusić semantyki słabsze przez argumenty funkcji `load`, `store`, `exchange`, czy `compare_exchange_weak` (patrz [tutaj](http://en.cppreference.com/w/cpp/atomic/memory_order)).

Sprawdź wynik następującego kodu (kod ten wykorzystuje metodę `operator++` specjalizacji `std::atomic<int>` szablonu `std::atomic`).

Przykład [atomic-volatile.cpp](atomic-volatile.cpp)

```cpp
#include <thread>
#include <iostream>
#include <atomic>

std::atomic<int> a{0};
volatile int v{0};

void f() {
    for (int i = 0; i < 1000000; i++) {
        a++;
        v++;
    }
}

int main() {
    std::cout << "main() starts" << std::endl;
    std::thread t1{f};
    std::thread t2{f};
    t1.join();
    t2.join();
    std::cout << "main() completes: v=" << v <<" a="<<a<<std::endl;
}
```

#### Zadanie
- Zamień modyfikację `a` przez `++` na `+=`.
- Oznacz zmienne synchronizacyjne w algorytmie Petersona jako `std::atomic`. Czy algorytm działa poprawnie?
- Uruchom algorytm Petersona na jednym rdzeniu `taskset -c 0 ./peterson`. Dlaczego program działa tak wolno?

## std::atomic do nieblokujących struktur danych

`std::atomic` udostępnia serię metod `compare_exchange(T& expected, T desired)`. Metody te atomowo: 

1. porównują `*this` z `expected i`, jeśli są takie same, 

2a. zamieniają `*this` na `desired`; a w przeciwnym wypadku 

2b. zamieniają `expected` na `this*`. 

Porównanie w `1.` opiera się na porównaniu bit po bicie reprezentacji obiektów `expected` i `this`.

Taki mechanizm pozwala na zrealizowanie nieblokujących struktur danych. Standardowo przed modyfikacją struktury danych powinniśmy ją zablokować (np. muteksem). Muteksy są jednak dosyć kosztowne, dlatego, jeśli konflikt występuje dosyć rzadko, a semantyka rozwiązania konfliktu jest jednoznaczna, lepiej wykorzystać operację `compare_exchange` wykonywaną w pętli.

`compare_exchange` występuje w dwóch podstawowych wariantach:

- słabym (`compare_exchange_weak`): zapis może nie nastąpić nawet gdy `*this` jest identyczne z  `expected`
- mocnym (`compare_exchange_strong`): zapis zawsze nastąpi gdy `*this` jest identyczne z `expected`.   Wariant mocny jest bardziej kosztowny

Następujący kod (zaadopotowany z http://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange) implementuje stos. Kod wykorzystuje `std::atomic<node<T>*>`, czyli specjalizację typu atomic dla wskaźnika.

Przykład [lockfree-stack.cpp](lockfree-stack.cpp)

```cpp
template<typename T>
struct node
{
    T data;
    node* next;
    node(const T& data) : data(data), next(nullptr) {}
};

template<typename T>
class stack
{
    std::atomic<node<T>*> head;

 public:
    void push(const T& data)
    {
      node<T>* new_node = new node<T>(data);
      // thread-unsafe code:
      // new_node->next = head;
      // head = new_node;

      // put the current value of head into new_node->next
      new_node->next = head.load();

      // now make new_node the new head, but if the head
      // is no longer what's stored in new_node->next
      // (some other thread must have inserted a node just now)
      // then put that new head into new_node->next and try again
      while(!head.compare_exchange_weak(new_node->next, new_node))
          ; // the body of the loop is empty
    }

    node<T>* get_head() {
        return head;
    }
};
```
#### Zadanie
Zamień kod na wariant niepoprawny (w komentarzu) i sprawdź różnice w działaniu.

## Zadanie punktowane: implementacja algorytmu Dekkera

**Zaimplementuj algorytm Dekkera.**

## Bibliografia

- You Don't Know Jack About Shared Variables or Memory Models, Hans-J. Boehm, Sarita V. Adve Communications of the ACM, Vol. 55 No. 2, Pages 48-54, http://dx.doi.org/10.1145/2076450.2076465 http://queue.acm.org/detail.cfm?id=2088916
- Scott Meyers, Effective Modern C++, O'Reilly 2015, Item 40
- http://en.cppreference.com/w/cpp/language/memory_model


Autor: Krzysztof Rządca

Created: 2017-12-04 Mon 11:44

[Emacs](http://www.gnu.org/software/emacs/) 25.1.1 ([Org](http://orgmode.org/) mode 8.2.10)

[Validate](http://validator.w3.org/check?uri=referer)
