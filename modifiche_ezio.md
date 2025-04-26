# Modifiche apportate alla versione Ezio di VanitySearch

## Ottimizzazione della compilazione
- Aggiornamento delle flag di compilazione per ottimizzare le prestazioni CPU e GPU.
- Utilizzo di `-march=native` e `-O3` per la compilazione CPU.
- Utilizzo di `-gencode=arch=compute_86,code=sm_86` per la compilazione GPU, ottimizzata per NVIDIA GeForce RTX 3080.

## Modifiche al Makefile
- Aggiornamento delle CXXFLAGS per includere `-march=native` e `-O3` per ottimizzare le prestazioni.
- Aggiornamento delle LFLAGS per includere `-flto` per l'ottimizzazione del linker.
- Modifica delle flag NVCC per utilizzare l'architettura GPU corretta per RTX 3080.

## Ottimizzazione della memoria GPU
- Utilizzo di memoria unificata (cudaMallocManaged) per `inputKey` per semplificare l'accesso ai dati tra host e device.

## Prossimi passi
- Analisi del kernel CUDA `comp_keys` per ulteriori ottimizzazioni.
- Implementazione di una cache LRU per le operazioni di hash per ridurre i calcoli ridondanti.