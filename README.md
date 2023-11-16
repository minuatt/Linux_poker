# Linux_poker
한림대학교 2022학년도 1학기 운영체제 미니프로젝트(시연영상 : https://www.youtube.com/watch?v=LBWu_CIs17E)
## 개요
&nbsp;한림대학교 2022학년도 1학기 운영체제 미니프로젝트로 소켓통신을 이용한 카드게임을 만들어보았다.
1. 서버와 클라이언트간 통신과 자식프로세스와 부모프로세스간 데이터 통신구현
2. 다중 클라이언트 기능 구현

## 실행 요약
&nbsp; 이 프로젝트는 서버와 클라인언트의 통신으로 구현한다. </br>
- [game_server]
```c
void main(){
    /*********************************************************
     1. 소켓 통신을 위한 셋팅 *********************************
    **********************************************************/
    int s_sock_fd, new_sock_fd;
    struct sockaddr_in s_address, c_address;
    int addrlen = sizeof(s_address);
    int check;
    int child_process_num = 0; // 서버에 접속한 클라이언트의 수
    
    s_sock_fd = socket(AF_INET, SOCK_STREAM, 0);// 서버 소켓 선언 (AF_INET: IPv4 인터넷 프로토콜, SOCK_STREAM: TCP/IP)
    if (s_sock_fd == -1){
        perror("socket failed: ");
        exit(1);
    }
```
- 서버와 클라이언트를 연결하는 코드로, 서버 소켓의 포트번호는 8080, IP주소는 127.0.0.1을 사용하여 통신한다.
```c
/*********************************************************
     2. IPC를 위한 셋팅 *************************************
    **********************************************************/
    pid_t pid_0, pid_1, pid_2; // player 0과 palyer1의 pid를 저장할 변수
    key_t key_p2c_0 = 11111, key_c2p_0 = 22222;
    key_t key_p2c_1 = 33333, key_c2p_1 = 44444;
    key_t key_p2c_2 = 55555, key_c2p_2 = 66666;
    int msqid_p2c_0, msqid_c2p_0, msqid_p2c_1, msqid_c2p_1, msqid_p2c_2, msqid_c2p_2;
    //Message queue 초기화 (clear up)
    if((msqid_p2c_0=msgget(key_p2c_0, IPC_CREAT|0666))==-1){exit(-1);}
    msgctl(msqid_p2c_0, IPC_RMID, NULL);
    if((msqid_c2p_0=msgget(key_c2p_0, IPC_CREAT|0666))==-1){exit(-1);}
    msgctl(msqid_c2p_0, IPC_RMID, NULL);
    if((msqid_p2c_1=msgget(key_p2c_1, IPC_CREAT|0666))==-1){exit(-1);}
    msgctl(msqid_p2c_1, IPC_RMID, NULL);
    if((msqid_c2p_1=msgget(key_c2p_1, IPC_CREAT|0666))==-1){exit(-1);}
    msgctl(msqid_c2p_1, IPC_RMID, NULL);
    if((msqid_p2c_2=msgget(key_p2c_2, IPC_CREAT|0666))==-1){exit(-1);}
    msgctl(msqid_p2c_2, IPC_RMID, NULL);
    if((msqid_c2p_2=msgget(key_c2p_2, IPC_CREAT|0666))==-1){exit(-1);}
    msgctl(msqid_c2p_2, IPC_RMID, NULL);
    
    // Message queue 생성
    if((msqid_p2c_0=msgget(key_p2c_0, IPC_CREAT|0666))==-1){exit(-1);}
    if((msqid_c2p_0=msgget(key_c2p_0, IPC_CREAT|0666))==-1){exit(-1);}
    if((msqid_p2c_1=msgget(key_p2c_1, IPC_CREAT|0666))==-1){exit(-1);}
    if((msqid_c2p_1=msgget(key_c2p_1, IPC_CREAT|0666))==-1){exit(-1);}
    if((msqid_p2c_2=msgget(key_p2c_2, IPC_CREAT|0666))==-1){exit(-1);}
    if((msqid_c2p_2=msgget(key_c2p_2, IPC_CREAT|0666))==-1){exit(-1);}

```
- 미니프로젝트의 2개의 클라이언트 접속이 아닌 3개의 클라이언트 접속을 위한 코드이다 </br>
자식-부모간 메시지 큐를 하나씩 추가 생성하였고, player 2 의 pid는 pid_2에 저장하였다.

```c 
struct gameInfo sending_info, receiving_info; // 데이터 주고 받기 위한 구조체 선언 (부모 프로세스가 사용)
    // 각 플레이어에게 전달할 초기 게임 정보 생성 및 전송
    printf("Set game info\n");
    make_cards(); // 카드 생성
    shuffle(cards, 52); // 카드 셔플
    printf("\n@@ cards setting---------------------------\n");
    top_card = 0; //  덱 가장 위에 있는 카드의 인덱스 표시

    //player 0에게 줄 첫 6장 카드 선택 및 전송
    sending_info.num_cards = 0;
    for (int i=0; i<6; i++){ 
        sending_info.cards[i]= cards[i];
        sending_info.num_cards ++;
        top_card += 1;
    }
    sending_info.open_card = cards[top_card+6];
    msgsnd(msqid_p2c_0, &sending_info, sizeof(struct gameInfo),0); 
```
- 처음 게임시작할 때 첫 6장의 카드 전송 부분이다. 플레이어가 총 3명임을 구현하여, 
플레이어가 두명이였을때의 코드와 달리 msqid_p2c_2에 카드를 전송하게 하였고, 플레이어 2가 낸 카드도 top_card로 갱신되게 코드를 완성하였다.

```c
 //게임 진행을 위해 자식 프로세스 생성
        int c_pid = fork();
        if(c_pid==0){ // 자식 프로세스
            if(0==child_process_num){
                child_proc(new_sock_fd,  child_process_num, msqid_p2c_0, msqid_c2p_0); // player 0을 담당할 자식 프로세스
                
            }
            else if(1==child_process_num){
                child_proc(new_sock_fd,  child_process_num,  msqid_p2c_1, msqid_c2p_1); // player 1을 담당할 자식 프로세스
            }
            else if(2==child_process_num){
                child_proc(new_sock_fd, child_process_num, msqid_p2c_2, msqid_c2p_2);// player 2을 담당할 자식 프로세스
            }
            
        }
        else{ // 부모 프로세스
            if (child_process_num==0){
                pid_0 = c_pid; // 첫번째 자식 프로세스의 pid 저장
            }
            else if (child_process_num==1){
                pid_1 = c_pid; // 두번째 자식 프로세스의 pid 저장
            }
            else if(child_process_num==2){
                pid_2 = c_pid;
            }
            child_process_num ++;
            close(new_sock_fd);// 부모 프로세스는 새로운 소켓을 유지할 필요 없음.
```
- 게임 진행을 자식 프로세스를 생성하는 과정이다. 클라이언트가 연결되면, 클라이언트의 자식프로세스를 fork()로 생성하고, 데이터는 player0,1,2의 pid에 저장한다.
3개의 클라이언트를 구현했으므로, child_process_num이 2 일때 player 2의 자식 프로세스를 생성하고, pid_2에 정보를 저장하도록 코드를 완성하였다.

```c
   if (child_process_num >=3){// 3명의 Client가 접속하면 게임 시작
                while(1){//반복되는 게임 동작
                    ////////////////////////////////////////////////////////////////////////
                    // Player 0의 차례//////////////////////////////////////////////////////
                    //////////////////////////////////////////////////////////////////////
                    kill(pid_0, SIGUSR1);
                    printf("Player 0's turn\n");
                    // 1. 현재 오픈카드 정보 전송
                    sending_info.open_card = open_card;
                    msgsnd(msqid_p2c_0, &sending_info, sizeof(struct gameInfo),0);
                    // 2. 플레이어 0의 게임 정보 수신
                    msgrcv(msqid_c2p_0, &receiving_info, sizeof(struct gameInfo), 0 ,0);
                    // 3. 플레이어 0의 게임 정보에 따른 행동
                    if (receiving_info.open_card.value == open_card.value && receiving_info.open_card.suit == open_card.suit){
                        //3-1. open 카드 정보가 같으면, 플레이어가 카드를 내려놓지 못한 것. --> 새로운 카드 전달.
                        for(int i=0; i<receiving_info.num_cards; i++){
                            sending_info.cards[i] = receiving_info.cards[i];
                        }
                        sending_info.cards[receiving_info.num_cards] = cards[top_card];
                        sending_info.num_cards = receiving_info.num_cards+1;
                        top_card ++;
                        msgsnd(msqid_p2c_0, &sending_info, sizeof(struct gameInfo),0); // 첫 6장의 카드 전송
                    }
                    else{
                        //3-2. open 카드 정보가 다르면, 플레이어가 카드를 내려놓은 것. --> 현재 open 카드 정보 업데이트
                        open_card = receiving_info.open_card;
                    }
```
- 먼저 if(child_process num>=3)은 총 3개의 클라이언트를 만들었으므로 서버가 클라이언트가 3개 들어올때까지 대기한다는 의미이다. 또한, 게임룰로는 player가 낸 정보가 open_card와 같다면 카드를 내지 못한것으로 간주하고, 새로운 카드를 전달하고, open_card와 정보가 다르면 카드를 내려 놓은것으로 open_card의 정보를 갱신하는 코드이다.
```c
if (receiving_info.num_cards == 0){
                        kill(pid_0, SIGINT); // 승리
                        kill(pid_1, SIGQUIT); // 패배
                        kill(pid_2, SIGQUIT);// 패배
                        printf("Player 1 Win\n");
                    }
                    if (receiving_info.num_cards > 20){
                        kill(pid_0, SIGQUIT); // 패배
                        kill(pid_1, SIGINT);// 승리
                        kill(pid_2, SIGINT); // 승리
                        printf("Player 1 and Player 2 Win\n");
                    }

                    if (top_card >= 52){
                        //카드 다씀, 무승부
                        kill(pid_0, SIGILL);
                        kill(pid_1, SIGILL);
                        kill(pid_2, SIGILL);
                    }
```
- 위 코드는 게임종료될 조건이고, player0의 남은 카드가 0이면 승리시그널과 함께 프린트문이 출력되고, player0의 남은카드가 20장이 넘어가면 player1과 player2에겐 승리시그널이, player0에겐 패배시그널이, top card를 모두 소진하면 무승부 시그널이 전달되도록 코드를 완성하였고, player1과 player2의 조건도 동일하게 완성하였다.

```c
//Signal handling 함수 정의
void my_turn(int sig){
    struct socket_msg msg;
    msg.flag = 0;
    sprintf(msg.text, "********************\n*******Its your turn\n********************\n");
    send_sock(child_client_sock, msg); // 안내문구 전송
}
void win_sig(int sig){
    struct socket_msg msg;
    msg.flag = -99;
    sprintf(msg.text, "You are the winner!\n");
    send_sock(child_client_sock, msg); // 게임 결과 전송
    close(child_client_sock); // socket close
    exit(0);
}
void lose_sig(int sig){
    struct socket_msg msg;
    msg.flag = -99;
    sprintf(msg.text, "You are the loser.\n");
    send_sock(child_client_sock, msg); // 게임 결과 전송
    close(child_client_sock); // socket close
    exit(0);
}
void tie_sig(int sig){
    struct socket_msg msg;
    msg.flag = -99;
    sprintf(msg.text, "Its tie....\n");
    send_sock(child_client_sock, msg); // 게임 결과 전송
    close(child_client_sock); // socket close
    exit(0);
}
```
- 위  코드는 승리, 패배, 무승부 시그널을 정의한것으로 각 프로세스의 게임 종료될 조건에 따라 클라이언트가 해당 print의 내용을 전달하게 된다.
```c
	void child_proc(int sock, int client_id, int msqid_p2c, int msqid_c2p){
    /* 매개변수
        0. 소켓fd
        1. client 번호 (혹은 player 번호)
        2. parent to child 메세지큐 id
        3. child to parent 메세지큐 id
    */
    printf("child(%d) joined.\n", client_id);
    struct socket_msg msg; // client와 정보를 주고 받을 구조체
    struct gameInfo player_info; // player의 게임정보
    struct gameInfo receiving_info; // 부모 프로세스로에게 전달받을 게임 정보
    child_client_sock = sock; // 시그널 사용을 위해 sock 전역변수에 저장

    signal(SIGINT, win_sig); // 승리 시그널
    signal(SIGQUIT, lose_sig); //패배 시그널
    signal(SIGILL, tie_sig); //무승부 시그널

    // 초기 정보 수신
    msgrcv(msqid_p2c, &player_info, sizeof(struct gameInfo), 0 , 0);
    // msg.flag=0 -> 안내 문구
    // msg.flag=1 -> 안내문구 + 입력요망
    msg.flag = 0;
    sprintf(msg.text,"You got six cards.\n");
    send(sock, (struct socket_msg*)&msg, sizeof(msg), 0);
    for (int i=0; i<player_info.num_cards; i++) {
        sprintf(msg.text, "%d:(%c,%d), ", i, player_info.cards[i].suit, player_info.cards[i].value);
        send_sock(sock, msg);
    }
    sprintf(msg.text, "\nGame will starts soon----------------.\n");
    send_sock(sock, msg);
```
- 위 코드는 사용자에게 안내해주는 안내문구 및 시그널을 보이게 해주는 코드로, 소켓통신으로 이루어져있다.

 ```c
  while(1){
        // 시그널 올 때까지 대기
        signal(SIGUSR1, my_turn);
        pause();
        // 부모 프로세스로부터 정보 수신 및 업데이트
        msgrcv(msqid_p2c, &receiving_info, sizeof(struct gameInfo), 0 ,0);
        player_info.open_card = receiving_info.open_card;

        // 현재 오픈 카드 정보 Client에게 송신
        sprintf(msg.text, "--------------------------------------------------\n");
        send_sock(sock, msg);
        msg.flag = 0;
        sprintf(msg.text, "Current Open Card: (%c,%d)\n", player_info.open_card.suit, player_info.open_card.value);
        send_sock(sock, msg);
       
        // 현재 내 카드 정보 Client에게 송신
        sprintf(msg.text, "Your Cards List\n");
        send_sock(sock, msg);
        for (int i=0; i<player_info.num_cards; i++) {
            sprintf(msg.text, "%d:(%c,%d), ", i, player_info.cards[i].suit, player_info.cards[i].value);
            send_sock(sock, msg);
        }
        sprintf(msg.text, "\n");
        send_sock(sock, msg);
        sprintf(msg.text, "--------------------------------------------------\n");
        send_sock(sock, msg);

        // 드롭할 카드 선택 요청 (client로 부터 입력받아야 함)
        int user_input;
        msg.flag = 1; // input flag
        sprintf(msg.text, "Select card index:");
        send_sock(sock, msg);
```
- 소켓통신으로, 시그널이 올 때까지 대기하고, 대기가 끝나 입력시그널이 오면 클라이언트로 부터 입력을 받은뒤  print문에 따라 출력문이 client에게 전송된다.
```c
 // *client의 입력결과 수신
        // *수신된 정보 부모 프로세스에게 전달
        // *부모 프로세스에게 게임진행 결과 전달받아서, player info 수정
        msg = receive_sock(sock); // 클라이언트 입력결과 수신
        user_input = msg.flag;
        if (player_info.cards[user_input].suit==player_info.open_card.suit || player_info.cards[user_input].value==player_info.open_card.value){
            //카드 드롭
            if (player_info.num_cards == 1){
                // user_input 0, num cards가 1일때 오류 발생하는거 방지. (my_info.cards 정리하며 오류 발생.)
                player_info.num_cards=0;
            }
            else{
                struct card temp = player_info.cards[msg.flag];
                for (int i=user_input; i < player_info.num_cards; i++){ // 카드 리스트에서 드롭할 카드 빼기
                    player_info.cards[i].value = player_info.cards[i+1].value;
                    player_info.cards[i].suit = player_info.cards[i+1].suit;
                }
                player_info.num_cards -= 1;
                player_info.open_card = temp;
                msg.flag = 0;
                sprintf(msg.text, "Card (%c,%d) is dropped\n", temp.suit, temp.value);
                send_sock(sock, msg); // 업데이트 된 정보 전송
            }
            msgsnd(msqid_c2p, &player_info, sizeof(struct gameInfo),0);
        }
        else{
            msg.flag = 0;
            sprintf(msg.text, "You cannot drop this card. You will have another card.");
            send_sock(sock, msg); // 업데이트 된 정보 전송
            // 업데이트 된 정보 전송
            msgsnd(msqid_c2p, &player_info, sizeof(struct gameInfo),0);
            // 새로운 카드가 포함된 정보 수신.
            msgrcv(msqid_p2c, &receiving_info, sizeof(struct gameInfo), 0 ,0);
            player_info.num_cards = receiving_info.num_cards;
            for (int i=0; i < player_info.num_cards; i++){ // 카드정보 갱신
                player_info.cards[i].value = receiving_info.cards[i].value;
                player_info.cards[i].suit = receiving_info.cards[i].suit;
                if(i==player_info.num_cards-1){
                    sprintf(msg.text, "Card (%c,%d) is added\n", player_info.cards[i].suit, player_info.cards[i].value);
                    send_sock(sock, msg); // 업데이트 된 정보 전송
                    }
                }
        }
        
        msg.flag = 0;
        sprintf(msg.text, "Your turn is over. Waiting for the next turn.\n--------------------------------------------------\n");
        send_sock(sock, msg); // 턴 종료 후 대기
    }
```
- 위 코드는 client로부터 입력결과를 수신하고, 수신한정보를 부모프로세스에 전달 및 부모프로세스는 전달받은 정보로 player정보를 수정한다.</br>
- 입력받은 값이 open_card value 또는 suit 정보와 같으면 card is dropped 문구가, 입력받은 값이 open_card 의 value 또는 suit와 같지 않으면 you cannot drop this card. You will have another card가 출력되고 입력과 출력까지 끝나면 턴이 종료되어 your turn is over. Waiting for the next turn을 출력하게 되며 해당 프로세스는 대기하게 된다

```c
  //게임 진행을 위해 필요한 함수 정의
struct socket_msg receive_sock(int sock){ // 소켓 수신 함수
    struct socket_msg msg;
    recv(sock, (struct socket_msg*)&msg, sizeof(msg),0);
    return msg;
}
void send_sock(int sock, struct socket_msg msg){ // 소켓 송신 함수
    send(sock, (struct socket_msg*)&msg, sizeof(msg), 0);
}
void make_cards(){ // 카드 정보 생성
    // make cards
    printf("Make cards");
	for (int i=0;i<13;i++) {
		cards[i].value=i%13+1;
		cards[i].suit='c';
	}
	for (int i=0;i<13;i++) {
		cards[i+13].value=i%13+1;
		cards[i+13].suit='d';
	}
	for (int i=0;i<13;i++) {
		cards[i+26].value=i%13+1;
		cards[i+26].suit='h';
	}
	for (int i=0;i<13;i++) {
		cards[i+39].value=i%13+1;
		cards[i+39].suit='s';
    }
    
    
    for (int i=0;i<52;i++) {
		printf("(%c,%d) ", cards[i].suit,cards[i].value);
	}
    

    printf("\n\n");
}
 ```
- 총 52장의 카드를 만들었고 value값에는 숫자가 suit값에는 char가 들어가며 value와 suit값에 따라 카드를 내려놓을지 말지를 정하게된다.
## 게임진행
&nbsp;client 3개가 다 접속할때까지 server는 대기하다가, 3개가 다 들어오는순간 게임이 시작된다.</br> 자식프로세스와 부모프로세스간 통신으로 입력하고 카드정보 확인하고 일치하면 내려놓고 일치하지않으면 한장씩 가지는 동작을 반복하고, 턴이 바뀔때마다 server와 도 통신을 하게된다.</br>위 코드와 같이 player2가 남은카드가 0장이 되었기 때문에 player2에게는 승리시그널이, 나머지 두 플레이어에게는 패배 시그널이 전달되면서 게임이 종료하게된다.
## 결론
&nbsp;프로젝트의 목적은 서버와 클라이언트간 통신과 자식프로세스와 부모프로세스간 데이터 통신 구현, 다중 클라이언트 기능을 구현함에있다.</br>
일상생활에서 원카드 게임을 할 때, 보통 3명이상 게임을 진행하였기 때문에 3명의 플레이어를 만들어보고 싶었고, 자식-부모간 메시지큐, 추가적인 pid 생성, 새로운 클라이언트의 시그널 및 카드정보 갱신등으로, 3명의 플레이어가 게임하는 것을 구현할 수 있었다. </br>
결과적으로 게임도 잘돌아가고, 마지막 시그널, 클라이언트와 서버간 통신도 잘이루어져 성공적이라고 할 수있다. </br>
앞으로 더 보완할 점으로는 조커카드를 만들어서 실제 게임하는것처럼 조커카드를 뽑은플레이어가 다른플레이어에게 불이익을 주고, 혹은 폭탄 카드를 만들어서 폭탄 카드를 뽑으면 카드를 5장 드로우하는 것등을 구현해보려했지만, 아직 통신에대해 완벽하게 이해하지 못한거같아 번번히 실패했지만, 기회가 된다면 미니프로젝트 코드로 해내지 못한 구현을 해보고싶다.



 
