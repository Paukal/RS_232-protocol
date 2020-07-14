#include "rs232.h"
#include <fcntl.h> / File control definitions / #include <netdb.h> / getprotobyname / #include "md5.h" 
#include "md5.c" 
#include <time.h> 
#include "linked_list.c"

//data lengths: int numLength= 15; int messLength= 160;

int main( int argc, char argv[] ) {
/ FORWARDING TO RS232:

        *uzkomentuotas md5 hash funkcionalumas
            2.dokumentacija
            3.pajungimas i pagr programa

            pavyzdys.c -> rs232.c -> rs232receiver.c
*/
system("/bin/stty -F /dev/rs232 115200 cs8 -parenb -parodd -cstopb -crtscts -ixon -ixoff -echo");
init_diamond(argc, &argv);

return(0);

}

void rs232_send_num (char sms_sender, char sms_text){

            /*============================================================================
            ______________________________________________________________________________
            NUMBER LENGTH || TEXT LENGTH || SENDER NUMBER ||    SMS TEXT    || data md5
            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                2 Bytes         2 Bytes      9-12 Bytes       0-160 Bytes       32 Bytes    
            ==============================================================================*/

            int SPort =open("/dev/rs232", O_WRONLY | O_NONBLOCK ); //open port wth SysCall
            if (SPort == -1)
            {
                perror("open_port: Unable to open /dev/rs232 - ");
                return -1;
            }

            short numLength= strlen(sms_sender); //phone number length
            short textLength= strlen(sms_text); //text length
            char numL[3]; sprintf(numL, "%d", numLength); //phone number length in str
            char textL[4]; sprintf(textL, "%d", textLength); // text length in str
            short numDig=2; short textDig=3;//lengths length for mem segmentation
            char* MD5= (char*)malloc(33); //hashing value


            //phone number testing              
            if(numLength==0){
                write(SPort, "Phone number error: number is 0\n\n", 33);
                return -1;
            }
            else if(numLength/10 ==0){
                numL[1]= numL[0];
                numL[0]= '0';
                numL[2]= '\0';
            }
            else if(numLength/100 ==0)
                numDig=2;
            else{
                write(SPort, "Phone number error: number out of Bounds\n\n", 42);
                return -1;
            }

            //text character testing
            if(textLength==0)
            write(SPort, "Text error: character number is 0\n\n", 35); 
            else if(textLength/10 ==0){
                textL[2]= textL[0];
                textL[0]=textL[1]=0;
                textL[3]= '\0';
            }
            else if(textLength/100 ==0){
                textL[2]= textL[1];
                textL[1]= textL[0];
                textL[0]= '0';
                textL[3]= '\0';
            }
            else if(textLength/1000 ==0)
                textDig=3;
            else{
                write(SPort, "Text error: character number out of Bounds\n\n", 44); //??
                return -1;
            }
            int allDig= numDig+textDig;

            char *string = malloc(strlen(sms_sender)+strlen(sms_text));
            strcpy(string, sms_sender);
            strcat(string, sms_text);
            //MD5_KEY(string, MD5); //MD5 algorithm for key
            //int MD5Length= 32;

            //char buffer[allDig+numLength+textLength+MD5Length];
            char buffer[allDig+numLength+textLength];

            buffer[0]='\0';
            strncpy(buffer, numL, sizeof(numL));
            strcat(buffer, textL);
            strcat(buffer, sms_sender);
            strcat(buffer, sms_text);
            //strcat(buffer, MD5);

            char buffLength[4];

            buffLength[0]= ((strlen(buffer))/100) + '0';
            buffLength[1]= ((strlen(buffer))/10%10) + '0';
            buffLength[2]= ((strlen(buffer))%10) + '0';
            buffLength[3]= '\0';

            printf("buffer size sending: string %s, int %d\n", buffLength, sizeof(buffer));
            printf("buffer: %s\n", buffer);

            write(SPort, buffLength, 3); //pradinis communication, siunciamas length
            write(SPort, buffer, (strlen(buffer)));
            write(SPort, "\n", 1);

            memset(buffer, 0, sizeof buffer);
            close(SPort);

            //waiting for response from receiver
            bool_response();

}

/void MD5_KEY(char str, char *MD5) { // MD5 algorithm

int n;
unsigned char digest[16];

MD5_CTX ctx;
MD5Init(&ctx);
MD5Update(&ctx, str, strlen(str));
MD5Final(digest, &ctx);

for (n = 0; n < 32; ++n) {
    printf("%02x", (unsigned int)digest[n]);
}
for (n = 0; n < 16; ++n) {
    snprintf(&(MD5[n*2]), 16*2, "%02x", (unsigned int)digest[n]);
}

printf("\nHash string: %s", str);
printf("\nMD5 hashing: %s\n", MD5);

}*/

void init_diamond( int argc, char *argv[] ){ //daemon init pid_t pid = 0; pid_t sid = 0;

pid = fork();// fork a new child process

if (pid < 0)
{
    printf("fork failed!\n");
    exit(1);
}

if (pid > 0)// its the parent process
{
    printf("pid of child process %d \n", pid);
    exit(0); //terminate the parent process succesfully
}

umask(0);//unmasking the file mode

sid = setsid();//set new session
if(sid < 0)
{
    exit(1);
}

init_socket(argc, &argv); //reading from pavyzdys.c

}

void init_socket( int argc, char *argv[] ){ //socket init

char protoname[] = "tcp";
struct protoent *protoent;
int enable = 1;
int server_sockfd, client_sockfd;
socklen_t client_len;
ssize_t nbytes_read;
struct sockaddr_in client_address, server_address;
unsigned short server_port = 12345u;

if (argc > 1) {
    server_port = strtol(argv[1], NULL, 10);
}

protoent = getprotobyname(protoname);
if (protoent == NULL) {
    perror("getprotobyname");
    exit(EXIT_FAILURE);
}

server_sockfd = socket(
    AF_INET,
    SOCK_STREAM,
    protoent->p_proto
    /* 0 */
);
if (server_sockfd == -1) {
    perror("socket");
    exit(EXIT_FAILURE);
}

if (setsockopt(server_sockfd, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable)) < 0) {
    perror("setsockopt(SO_REUSEADDR) failed");
    exit(EXIT_FAILURE);
}

server_address.sin_family = AF_INET;
server_address.sin_addr.s_addr = htonl(INADDR_ANY);
server_address.sin_port = htons(server_port);
if (bind(
        server_sockfd,
        (struct sockaddr*)&server_address,
        sizeof(server_address)
    ) == -1
) {
    perror("bind");
    exit(EXIT_FAILURE);
}

if (listen(server_sockfd, 5) == -1) {
    perror("listen");
    exit(EXIT_FAILURE);
}
fprintf(stderr, "listening on port %d\n", server_port);

short num=0, text=0;
short sending=0;
char numArray[numLength], messArray[messLength];
short messNum=0; short messSend=0;

char *notif= (char *) malloc(4);

while(1){
    client_len = sizeof(client_address);
    client_sockfd = accept(
        server_sockfd,
        (struct sockaddr*)&client_address,
        &client_len
    );


if((nbytes_read = read(client_sockfd, notif, 4)) > 0){ //incomming sms sygnal

messNum++;

insertLast(messNum,"\0","\0");
struct node *MESSAGE= find(messNum);

MESSAGE->buffer= initBuffer();
char *buffer= MESSAGE->buffer;

if((nbytes_read = read(client_sockfd, buffer, BUFSIZ)) > 0){

        if(strcmp(buffer, "#")>0){
            num=1;
            nbytes_read = read(client_sockfd, buffer, BUFSIZ);

            memcpy(numArray, &buffer[0], nbytes_read);
            numArray[nbytes_read] = '\0';
        }

        if(num==1){
            num = 0;
            //write(STDOUT_FILENO, buffer, nbytes_read);
            nbytes_read = read(client_sockfd, buffer, BUFSIZ);
            nbytes_read = read(client_sockfd, buffer, BUFSIZ);
        }

        if(strcmp(buffer, "*")>0){
            text=1;
            nbytes_read = read(client_sockfd, buffer, BUFSIZ);

            memcpy(messArray, &buffer[0], nbytes_read);
            messArray[nbytes_read] = '\0';
        }

        if(text==1){
            sending= 1;
            text = 0;
            //write(STDOUT_FILENO, buffer, nbytes_read);
        }

        printf("\n---------------------------------------------\n");
        printf("sending number: %s\n", numArray);
        printf("sending message: %s\n", messArray);
        printf("message num: %d\n", messNum);

        setValues(messNum, numArray, messArray); //set values for sms
}
}

    if(sending== 1){
        sending=0;
        struct node *Link= find(++messSend);

        pid_t pid = fork();
        if (pid == 0)
        {
            // child process
            rs232_send_num(Link->data, Link->data2);
            exit(0);
        }
    }

}

return EXIT_SUCCESS;

}

void bool_response(){ //5sec delay

char *RESP_MES= (char*)malloc(2);

pid_t pid = fork();

if (pid == 0)
{
    int SPort =open("/dev/rs232", O_RDWR | O_NONBLOCK); //open port wth SysCall
    if (SPort == -1)
    {
        perror("open_port: Unable to open /dev/rs232 - ");
        return -1;
    }

    printf("\nWaiting for response:");

    time_t begin, end;
    double time_spent= 0;

    time(&begin);

    while(time_spent < 5){

        if(read(SPort, RESP_MES, 2)>1){
            if(strcmp(RESP_MES, "0.") == 0){
                printf("\nSUCCESS: Approved by receiver: %s\n", RESP_MES);
                printf("---------------------------------------------\n");
                write(SPort, "Communication successful", 25);
                write(SPort, "\n", 1);

                close(SPort);
                exit(0);
            }
            /*else if(strcmp(RESP_MES, "1.") == 0){
                printf("\nERROR: different hashes: %s\n", RESP_MES);
                printf("---------------------------------------------\n");
                write(SPort, "ERROR: different hashes", 24);
                write(SPort, "\n", 1);

                close(SPort);
                exit(0);
            }*/
            else {
                printf("\nERROR: unindentified response: %s\n", RESP_MES);
                printf("---------------------------------------------\n");
                write(SPort, "ERROR: received unindentified response", 39);
                write(SPort, "\n", 1);

                close(SPort);
                exit(0);
            }
        }

        time(&end);
        time_spent= difftime(end, begin);
    }

    printf("\nERROR: RESPONSE TIMED OUT\n");
    printf("---------------------------------------------\n");
    write(SPort, "ERROR: sender hasnt received message confirmation", 50);
    write(SPort, "\n", 1);

    close(SPort);
    exit(0);
}

}
