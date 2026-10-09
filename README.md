# weryfikacja-kluczy-publicznych-i-podpis-w-GPG
Cel zadania
W tym ćwiczeniu każdy z was podpisze dokument i zweryfikuje dokument partnera. Sprawdzicie trzy różne wyniki: brak potrzebnego klucza publicznego, poprawny podpis oraz podpis niezgodny ze zmienioną treścią. Porównacie też pełne odciski kluczy.

Pracujcie na danych testowych. Przekazujcie sobie tylko klucze publiczne, dokumenty i podpisy. Klucza prywatnego ani hasła do niego nie udostępniajcie nikomu.

Przygotowanie. Wykonują obie osoby
Otwórzcie terminale na swoich maszynach. Każde z was wykonuje ten blok osobno i pozostawia ten sam terminal otwarty przez całe ćwiczenie:

gpg --version
LAB_DIR="$(mktemp -d "$PWD/learnit-gpg.XXXXXX")"
mkdir -p "$LAB_DIR/gpg" "$LAB_DIR/outbox" "$LAB_DIR/inbox"
chmod 700 "$LAB_DIR/gpg"
export GNUPGHOME="$LAB_DIR/gpg"
export GPG_TTY="$(tty)"
printf 'Mój katalog: %s\n' "$LAB_DIR"
outbox służy do przygotowania plików dla partnera, a inbox do zapisania otrzymanych plików. Osobny katalog gpg daje każdej osobie czysty profil bez starych kluczy.

Osoba A wykonuje tylko poniższy blok:

MY_NAME='Alicja Lab'
MY_EMAIL='alicja@lab.invalid'
PARTNER_EMAIL='bob@lab.invalid'
Osoba B wykonuje zamiast tego tylko ten blok:

MY_NAME='Bob Lab'
MY_EMAIL='bob@lab.invalid'
PARTNER_EMAIL='alicja@lab.invalid'
Teraz obie osoby tworzą własny klucz do podpisów i eksportują jego część publiczną:

gpg --quick-generate-key "$MY_NAME <$MY_EMAIL>" ed25519 sign 1y
gpg --fingerprint "$MY_EMAIL"
gpg --armor --export "$MY_EMAIL" > "$LAB_DIR/outbox/public.asc"
GnuPG może poprosić o hasło chroniące klucz prywatny. Zapamiętaj je, ale nie przesyłaj partnerowi. Pełny odcisk wyświetlony przez --fingerprint będzie potrzebny przy wymianie plików.

Etap 1. Osoba A podpisuje dokument
Na początek A jest autorem, a B weryfikatorem. A wykonuje:

cat > "$LAB_DIR/outbox/raport.txt" <<EOF
Autor: $MY_NAME
Ćwiczenie: weryfikacja podpisów GPG
Zatwierdzam wersję testową raportu
EOF

gpg --armor --local-user "$MY_EMAIL" \
  --output "$LAB_DIR/outbox/raport.txt.sig.asc" \
  --detach-sign "$LAB_DIR/outbox/raport.txt"
A przesyła B dokładnie te trzy pliki z własnego outbox:

Plik	Co zawiera?
public.asc	Klucz publiczny autora.
raport.txt	Oryginalna, czytelna treść.
raport.txt.sig.asc	Podpis oddzielony od treści.
B zapisuje wszystkie trzy pliki w swoim katalogu inbox pod dokładnie takimi nazwami. Możecie je wymienić sposobem uzgodnionym na zajęciach. A dodatkowo odczytuje B pełny odcisk klucza na głos albo przekazuje go inną drogą niż plik public.asc.

Etap 2. Osoba B sprawdza klucz i podpis
B najpierw próbuje weryfikacji bez importu klucza A:

gpg --verify "$LAB_DIR/inbox/raport.txt.sig.asc" \
  "$LAB_DIR/inbox/raport.txt"
Spodziewany wynik: Can't check signature: No public key. To znaczy, że w tym profilu nie ma jeszcze potrzebnego klucza. Nie jest to wynik „niepoprawny podpis”.

B wyświetla odcisk z otrzymanego pliku public.asc i porównuje całą wartość z odciskiem podanym przez A innym sposobem:

gpg --show-keys --fingerprint "$LAB_DIR/inbox/public.asc"
Jeżeli odciski się różnią, zatrzymajcie ćwiczenie. Sprawdźcie, czy wymieniliście właściwy plik. Nie polegajcie wyłącznie na nazwie „Alicja Lab” lub adresie widocznym przy kluczu.

Gdy odciski są zgodne, B importuje klucz i powtarza weryfikację:

gpg --import "$LAB_DIR/inbox/public.asc"
gpg --fingerprint "$PARTNER_EMAIL"
gpg --verify "$LAB_DIR/inbox/raport.txt.sig.asc" \
  "$LAB_DIR/inbox/raport.txt"
Spodziewany wynik: Good signature. Może pojawić się również ostrzeżenie, że GnuPG nie ma lokalnego potwierdzenia tożsamości właściciela klucza. Zwróćcie uwagę, że to osobna kwestia: porównaliście odcisk z partnerem, ale program nie wie automatycznie, jak to zrobiliście.

Do zanotowania: Co sprawdza Good signature? Skąd wiecie, że użyliście klucza, którego odcisk podała A?

Etap 3. Osoba B zmienia kopię dokumentu
B tworzy osobną wersję pliku, w której zmienia jedno słowo. Nie zmienia otrzymanego podpisu ani oryginału:

sed 's/Zatwierdzam/Nie zatwierdzam/' \
  "$LAB_DIR/inbox/raport.txt" \
  > "$LAB_DIR/inbox/raport-zmieniony.txt"

gpg --verify "$LAB_DIR/inbox/raport.txt.sig.asc" \
  "$LAB_DIR/inbox/raport-zmieniony.txt"
Spodziewany wynik: BAD signature. Teraz B sprawdza ten sam podpis z oryginalnym dokumentem:

gpg --verify "$LAB_DIR/inbox/raport.txt.sig.asc" \
  "$LAB_DIR/inbox/raport.txt"
Spodziewany wynik: Ponownie Good signature.

Do zanotowania: Dlaczego ten sam podpis daje dwa różne wyniki? Czy podpis ukrywa treść raport.txt?

Etap 4. Zamieńcie się rolami
Teraz B jest autorem, a A weryfikatorem. B wykonuje blok z Etapu 1 na swojej maszynie, używając swoich zmiennych MY_NAME i MY_EMAIL, po czym przekazuje A trzy pliki ze swojego outbox. B podaje A swój pełny odcisk innym sposobem niż przesłanie klucza.

A zapisuje pliki B w swoim inbox i wykonuje bloki z Etapów 2 i 3. Dzięki oddzielnym profilom A nie ma jeszcze klucza publicznego B, choć posiada własny klucz prywatny. Każde z was powinno raz otrzymać wynik Good signature dla dokumentu partnera i BAD signature dla zmienionej kopii.

Gdy wynik jest inny niż oczekiwany
No public key po imporcie: Upewnij się, że pracujesz we właściwym terminalu i profilu oraz że importowałeś klucz osoby, która podpisała dokument. Sprawdź pełny odcisk.
BAD signature dla oryginału: Sprawdź, czy do pliku nie dodano spacji, nowej linii lub innych zmian podczas przesyłania. Podpis musi dotyczyć dokładnie tych bajtów, które weryfikujesz.
Nie można utworzyć podpisu: Sprawdź, czy w bieżącym profilu masz własny klucz prywatny i czy MY_EMAIL wskazuje ten klucz. GnuPG może potrzebować działającego gpg-agent oraz programu do wpisania hasła.
Nie wiesz, gdzie są pliki: Wyświetl zawartość swoich katalogów i sprawdź wartości zmiennych:
printf 'Katalog: %s\nMój klucz: %s\nKlucz partnera: %s\n' \
  "$LAB_DIR" "$MY_EMAIL" "$PARTNER_EMAIL"
ls -l "$LAB_DIR/outbox" "$LAB_DIR/inbox"
gpg --list-secret-keys
gpg --list-keys
Wyniki do oddania
Przygotujcie wspólną notatkę, maksymalnie jedną stronę. Nie dołączajcie kluczy prywatnych ani haseł.

Test	Wynik GnuPG lub obserwacja	Co z tego wynika?
Sprawdzenie przed importem klucza autora		
Oryginał i właściwy klucz publiczny		
Zmieniony dokument, ten sam podpis		
Ponowna weryfikacja oryginału		
Porównanie pełnego odcisku klucza partnera		
Odpowiedzcie własnymi słowami:

Dlaczego Good signature samo w sobie nie potwierdza tożsamości realnej osoby?
Czym różni się No public key od BAD signature?
Co zrobicie, jeśli pełny odcisk otrzymanego klucza różni się od odcisku potwierdzonego przez partnera?
Materiał pomocniczy: oficjalny podręcznik GnuPG.
