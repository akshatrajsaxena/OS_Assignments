# OS_Assignments
#include "loader.h"
Elf32_Ehdr *ehdr;
Elf32_Phdr *phdr;
int fd;
void loader_cleanup() {
     
        free(ehdr);
        free(phdr);
        if (fd >= 0) {
            close(fd);
            fd = -1;

    }

}
void load_and_run_elf(char** exe) {

    fd = open(exe[1], O_RDONLY);

    if (fd == -1) {

        perror("Error opening ELF file");

        return;

    }

    ehdr = (Elf32_Ehdr*)malloc(sizeof(Elf32_Ehdr));

    if (read(fd, ehdr, sizeof(Elf32_Ehdr)) != sizeof(Elf32_Ehdr)) {

        perror("Error reading ELF header");

        return;

    }

    phdr = (Elf32_Phdr*)malloc(ehdr->e_phentsize * ehdr->e_phnum);

    lseek(fd, ehdr->e_phoff, SEEK_SET);

    if (read(fd, phdr, ehdr->e_phentsize * ehdr->e_phnum) !=

        ehdr->e_phentsize * ehdr->e_phnum) {

        perror("Error reading program header table");

        return;

    }


    for (int i = 0; i < ehdr->e_phnum; ++i) {

        if (phdr[i].p_type == PT_LOAD) {

            void* load_addr = (void*)(uintptr_t)phdr[i].p_vaddr;

            void* mem = mmap(load_addr, phdr[i].p_memsz,

                             PROT_READ | PROT_WRITE | PROT_EXEC,

                             MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

            lseek(fd, phdr[i].p_offset, SEEK_SET);

            if (read(fd, mem, phdr[i].p_filesz) != phdr[i].p_filesz) {

                perror("Error reading segment content");

                return;

            }

        }

    }




    int (*_start)() = (int (*)())(uintptr_t)ehdr->e_entry;

    int result = _start();

    printf("User _start return value = %d\n", result);

}



int main(int argc, char** argv) {

    if (argc != 2) {

        printf("Usage: %s <ELF Executable> \n", argv[0]);

        exit(1);

    }

    load_and_run_elf(argv);

    loader_cleanup();

    return 0;

}
