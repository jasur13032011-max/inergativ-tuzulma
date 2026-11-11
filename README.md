# inergativ-tuzulma
Barcha o'tilgan mavzularni (dekoratorlar, generatorlar, context manager'lar, comprehensions va aniq tiplashtirish) o'z ichiga olgan yagona kutubxona qidiruv tizimi moduli:

```python
"""
Kutubxona qidiruv tizimi - Integrativ takrorlash namunasi.
Ushbu modul `mypy` tip tekshiruvchisidan toza o'tadi.
"""

from typing import List, Dict, Set, Optional, Iterator
from functools import lru_cache


# ==============================================================================
# 1. Context Manager (Kutubxona resursi)
# ==============================================================================

class Library:
    """Kutubxona resursini boshqarish uchun context manager klassi."""

    def __init__(self, books: List[Dict[str, any]]) -> None:
        self.books: List[Dict[str, any]] = books

    def __enter__(self) -> List[Dict[str, any]]:
        print("Kutubxona ochildi: resurslar tayyor.")
        return self.books

    def __exit__(self, exc_type, exc_val, exc_tb) -> None:
        print("Kutubxona yopildi: resurslar tozalandi.")


# ==============================================================================
# 2. Generator (Dangasa - lazy ishlab chiqarish)
# ==============================================================================

def get_available_books(books: List[Dict[str, any]]) -> Iterator[Dict[str, any]]:
    """Faqat mavjud (available == True) kitoblarni birma-bir yield qiluvchi generator."""
    for book in books:
        if book.get("available", False):
            yield book


# ==============================================================================
# 3. Keshzlangan sarlavha qidiruvi (@lru_cache) va Optional tip
# ==============================================================================

# Eslatma: lru_cache obyektlarni (masalan, list) yoki generatorlarni keshlaganda 
# ehtiyot bo'lish kerak. Quyida qidiruv faqat o'zgarmas (immutable) sarlavha 
# matnini qabul qilib, natijani qaytaradi.
@lru_cache(maxsize=128)
def search_book_by_title_cached(title: str, books_tuple: tuple) -> Optional[str]:
    """
    Berilgan sarlavha bo'yicha kitobni qidiradi.
    Topilsa muallifini, topilmasa None qaytaradi.
    """
    for book in books_tuple:
        if book["title"].lower() == title.lower():
            return book["author"]
    return None


# ==============================================================================
# 4. Tuzoq (Trap) namoyishi: Generatorga @lru_cache qo'llash xatosi
# ==============================================================================

@lru_cache(maxsize=128)
def cached_generator_trap(books_tuple: tuple) -> Iterator[Dict[str, any]]:
    """
    DIQQAT (Tuzoq): Generator funksiyaga @lru_cache qo'shish xato hisoblanadi.
    Chunki kesh generator obyektining holatini (state) saqlab qoladi. 
    Ikkinchi marta chaqirilganda generator allaqachon tugagan (exhausted) 
    obyektni qaytaradi va hech qanday natija bermaydi.
    """
    for book in books_tuple:
        yield book


# ==============================================================================
# 5. Demo va Comprehensions
# ==============================================================================

def main() -> None:
    # Boshlang'ich ma'lumotlar to'plami
    raw_books = [
        {"title": "Python asoslari", "author": "Anvar Narzullayev", "available": True},
        {"title": "Algoritmlar", "author": "Otabek", "available": False},
        {"title": "Toza Kod", "author": "Robert Martin", "available": True},
        {"title": "Sun'iy Intellekt", "author": "Anvar Narzullayev", "available": True},
    ]

    # Kutubxona context manager'ini ishlatish
    with Library(raw_books) as books:
        print("\n--- 1. Mavjud kitoblar (Generator orqali) ---")
        # Generator yordamida faqat mavjud kitoblarni chiqarish
        for book in get_available_books(books):
            print(f"Kitob: '{book['title']}', Muallif: {book['author']}")

        print("\n--- 2. Set Comprehension (Noyob mualliflar) ---")
        # Kitoblar ro'yxatidan noyob mualliflarni yig'ish uchun set comprehension
        unique_authors: Set[str] = {book["author"] for book in books}
        print(f"Noyob mualliflar: {unique_authors}")

        print("\n--- 3. @lru_cache yordamida qidiruv ---")
        # Tuple ga o'tkazamiz (lru_cache uchun hashlanuvchi bo'lishi shart)
        books_tuple = tuple(books)
        
        author = search_book_by_title_cached("Toza Kod", books_tuple)
        print(f"'Toza Kod' kitobining muallifi: {author}")

        # Keshdan olinishini tekshirish uchun takroriy qidiruv
        author_cached = search_book_by_title_cached("Toza Kod", books_tuple)
        print(f"Takroriy qidiruv (keshdan): {author_cached}")

        print("\n--- 4. Generatorga @lru_cache qo'llash tuzog'i ---")
        gen_1 = cached_generator_trap(books_tuple)
        print("1-marta chaqiruv natijasi:")
        for b in gen_1:
            print(b["title"])

        gen_2 = cached_generator_trap(books_tuple)
        print("2-marta chaqiruv natijasi (Kesh tufayli bo'sh qaytadi, chunki generator tugagan):")
        items_count = sum(1 for _ in gen_2)
        print(f"Ikkinchi chaqiruvdagi elementlar soni: {items_count} (Tuzoq!)")


if __name__ == "__main__":
    main()

```

### Asosiy talablar va tushuntirishlar:

1. **`Library` Context Manager**: `__enter__` metodi resurs ochilganini xabar qilib kitoblar ro'yxatini qaytaradi, `__exit__` metodi esa blokdan chiqishda resurs yopilganini konsolga chiqaradi.
2. **Generator (`get_available_books`)**: `yield` kalit so'zi yordamida faqat `available == True` bo'lgan kitoblarni xotirani tejagan holda birma-bir berib boradi.
3. **Kesh va Optional**: `@lru_cache` bilan bezatilgan `search_book_by_title_cached` funksiyasi qidiruv natijasini keshlaydi. Agar kitob topilmasa, aniq `Optional[str]` tipiga muvofiq ravishda `None` qaytaradi.
4. **Set Comprehension**: `{book["author"] for book in books}` qatori yordamida takrorlanmaydigan (noyob) mualliflar to'plami bir qatorda yig'ib olindi.
5. **Generator + `lru_cache` Tuzog'i**: Generator funksiyalarga `@lru_cache` qo'llanilganda, Python keshda yaratilgan generator obyektining o'zini saqlab qoladi. Ikkinchi marta chaqirilganda generator avvalgi safar oxirigacha o'qib bo'lingani uchun bo'sh (`exhausted`) qaytadi. To'g'ri yechim sifatida generatorlarga emas, balki aniq natija (qiymat yoki tuple/list) qaytaruvchi funksiyalarga kesh qo'llash kerak.
