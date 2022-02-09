import subprocess
import threading
import json
import socket
import select
import random
import base64
import time
import os


PORT = 12345
UI_COLORS = {"CGREEN": '\33[32m', "BLUE": '\x1b[1;37;44m', "LIGHT": '\x1b[7;37;40m', "RED": '\x1b[1;37;41m',
             "GREEN": '\x1b[5;30;42m', "YELLOW": '\x1b[2;30;43m', "PINK": '\x1b[2;30;45m', "END": '\x1b[0m'}
NAME = ""
MY_IP = str(subprocess.check_output(
    "hostname -I", shell=True), 'utf-8').split(' ')[0]
TARGET_IPS = MY_IP+"/24"
ONLINES = {}
BUFFER_SIZE = 10240
CHUNK_SIZE = 1500
ACKNOWLEDGED_DATA = {}
RWND = 10


def print_onlines(option=""):
    """
    If option is "only print", prints online users.
    If option is not provided, prints online users and waits for user input to send something.
    """
    if(len(ONLINES) == 0):
        print(UI_COLORS["BLUE"]+"NO ONE ELSE IS ONLINE"+UI_COLORS["END"])
    else:
        print("\n\n"+UI_COLORS["BLUE"]+"ONLINE USERS:"+UI_COLORS["END"])
        i = 1
        for key in ONLINES:
            print("{} {} :  {} {}".format('\x1b[1;{};40m'.format(
                30+i % 8), i, key, UI_COLORS["END"]))
            i = i+1
        print()
        if option == "only print":
            print(UI_COLORS["LIGHT"] +
                  "WHO DO YOU WANT TO MESSAGE?"+UI_COLORS["END"]+"\t")
        else:
            send_something_thread = threading.Thread(target=send_something)
            send_something_thread.start()


def send_discover():
    """
    Broadcasts TYPE 1 discover message 10 times with random id.
    """
    ID = random.randint(0, 10e7)
    for _ in range(10):
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
            sock.bind(("", 0))
            sock.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
            message = b'{"TYPE":1, "NAME":"'+bytearray(NAME.encode())+b'"'+b', "IP":"'+bytearray(
                MY_IP.encode())+b'", "ID":'+str(ID).encode()+b'}'
            sock.sendto(message, ("<broadcast>", 12345))


def send_discover_response(target_ip):
    """
    Sends TYPE 2 discover response message to the target ip.
    """
    message = b'{"TYPE":2, "NAME":"' + \
        bytearray(NAME.encode())+b'"'+b', "IP":"' + \
        bytearray(MY_IP.encode())+b'"}'
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.connect((target_ip, PORT))
            s.sendall(message)
    except:
        print("failed due to timeout")


def send_something():
    """
    Calls another functions to send a simple message or file. 
    """
    while True:
        target_name = input(
            UI_COLORS["LIGHT"] + "WHO DO YOU WANT TO MESSAGE?"+UI_COLORS["END"]+"\t")
        target_ip = ONLINES.get(target_name)
        if(target_ip is None):
            print(UI_COLORS["RED"] + "*** WRONG INPUT ***" + UI_COLORS["END"])
        else:
            break
    while True:
        message_type = input(
            UI_COLORS["LIGHT"] + "\nDo you want to send a simple message or a file? (1 for message, 2 for file)\t"+UI_COLORS["END"])
        if message_type == "1":
            send_chat_message(target_name, target_ip)
            break
        elif message_type == "2":
            send_file(target_ip)
            break
        else:
            print(UI_COLORS["RED"] + "*** WRONG INPUT ***" + UI_COLORS["END"])


def send_chat_message(target_name, target_ip):
    """
    Sends simple chat message to target ip. After sending, asks for "who do you want to chat?" question again.
    """
    input_message = input(
        UI_COLORS["YELLOW"]+"YOUR MESSAGE:\t" + UI_COLORS["END"])
    message = b'{"TYPE":3, "NAME":"'+bytearray(
        NAME.encode())+b'"'+b', "BODY":"'+bytearray(input_message.encode())+b'"}'
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.connect((target_ip, PORT))
            s.sendall(message)

    except Exception as ex:
        print(ex)
        print(
            UI_COLORS["PINK"]+"IT SEEMS THAT USER YOU ARE TRYING TO REACH IS OFFLINE"+UI_COLORS["END"])
        ONLINES.pop(target_name)
    print_onlines()


def send_file(target_ip):
    """
    Sends TYPE 4 message, file, to target ip.
    It first lists the files in the same directory, then captures input.
    Reads the file 1500 byte at a time and sends to target user with UDP.
    Waits acknowledgement for 1 second. If no acknowledgement came, resends until one come.    
    Prompts completion of file sending.    
    """
    print("Files in this directory: " + ', '.join(os.listdir()))
    file = input(
        UI_COLORS["YELLOW"]+"Which file do you want to send?\t" + UI_COLORS["END"])
    ACKNOWLEDGED_DATA[file] = 0
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as sock:
        sock.bind(("", 0))
        try:
            with open(file, 'rb') as infile:
                i = 1
                while True:
                    if ACKNOWLEDGED_DATA[file] == i-1:
                        chunk = bytearray(infile.read(CHUNK_SIZE))
                        chunk_encoded = base64.b64encode(chunk)
                        i += 1
                    if not chunk:  # send empty data to tell that file transfer is completed
                        message = b'{"TYPE":4, "NAME":"'+bytearray(file.encode()) + b'"'+b',  "SEQ":' + bytearray(
                            str(-1).encode())+b', "DATA":"", "IP": "'+bytearray(MY_IP.encode())+b'"}'
                        sock.sendto(message, (target_ip, 12345))
                        prev_time = time.time()
                        while str(ACKNOWLEDGED_DATA[file]) != str(-1):
                            if time.time()-prev_time > 1:
                                sock.sendto(message, (target_ip, 12345))
                                prev_time = time.time()
                        break
                    message = b'{"TYPE":4, "NAME":"'+bytearray(file.encode()) + b'"'+b',  "SEQ":' + bytearray(
                        str(i-1).encode())+b', "DATA":"'+chunk_encoded+b'", "IP": "'+bytearray(MY_IP.encode())+b'"}'
                    sock.sendto(message, (target_ip, 12345))
                    prev_time = time.time()
                    while str(ACKNOWLEDGED_DATA[file]) != str(i-1):
                        if time.time()-prev_time > 1:
                            sock.sendto(message, (target_ip, 12345))
                            prev_time = time.time()
            print(UI_COLORS["CGREEN"] +
                  "\nFile sent successfuly"+UI_COLORS["END"])
            print_onlines()
        except Exception as e:
            print(e)
            print("There is no files exist with that name!")
            print_onlines()


def send_acknowledgement(target_ip, name, seq):
    """
    Sends TYPE5, acknowledgement message to the target ip with given sequence of given file name.
    """
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.connect((target_ip, PORT))
            message = b'{"TYPE":5, "NAME": "'+bytearray(name.encode())+b'", "SEQ":' + bytearray(
                str(seq).encode())+b', "RWND":"'+bytearray(str(RWND-1).encode())+b'"}'
            s.sendall(message)
    except Exception as e:
        # print(e)
        ...


def initialize_tcp_server():
    """
    This is the TCP socket.
    It waits for TYPE 2,3,5 messages
    It first captures the data came and converts it to json.
    Then calls necessary functions.
    """
    while True:
        #print("server started")
        try:
            with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                s.bind((MY_IP, PORT))
                s.listen()  # listen
                conn, addr = s.accept()
                with conn:
                    if addr[0] == MY_IP:
                        break
                    #print("Connected by", addr)
                    msg = ""
                    while True:
                        part_of_data = str(conn.recv(BUFFER_SIZE), 'utf-8')
                        # print(repr(data))
                        msg += part_of_data
                        if not part_of_data:
                            break
                    # print(data)
                    if msg == "":
                        break

            incoming_message_in_json = json.loads(msg)
            # add them to onlines list
            if incoming_message_in_json["TYPE"] == 2:
                if(ONLINES.get(incoming_message_in_json["NAME"]) is None):
                    ONLINES[incoming_message_in_json["NAME"]
                            ] = incoming_message_in_json["IP"]
                    print_onlines()
            elif incoming_message_in_json["TYPE"] == 3:  # message came
                if (ONLINES.get(incoming_message_in_json["NAME"]) is None):
                    # new user, add it to onlines list.
                    print(UI_COLORS["GREEN"] +
                          "FOUND A NEW USER!"+UI_COLORS["END"])
                    print(
                        "\n\n"+UI_COLORS["GREEN"]+"*** YOU HAVE A NEW MESSAGE ***"+UI_COLORS["END"])
                    print(UI_COLORS["CGREEN"] + incoming_message_in_json["NAME"] +
                          ": " + UI_COLORS["END"] + incoming_message_in_json["BODY"])
                    ONLINES[incoming_message_in_json["NAME"]] = addr[0]
                    print_onlines()
                else:
                    # user was already in our list
                    print(
                        "\n\n"+UI_COLORS["GREEN"]+"*** YOU HAVE A NEW MESSAGE ***"+UI_COLORS["END"])
                    print(UI_COLORS["CGREEN"] + incoming_message_in_json["NAME"] +
                          ": " + UI_COLORS["END"] + incoming_message_in_json["BODY"])
                    print_onlines("only print")
            # acknowledment came for file transfer
            elif incoming_message_in_json["TYPE"] == 5:
                ACKNOWLEDGED_DATA[incoming_message_in_json["NAME"]
                                  ] = incoming_message_in_json["SEQ"]
                if incoming_message_in_json["SEQ"] == "-1":
                    print_onlines()

        except Exception as e:
            # print(e)
            pass


def initialize_udp_server_listen():
    """
    This is the UDP socket.
    It waits for TYPE 1,4 messages
    It first captures the data came and converts it to json.
    Then calls necessary functions.
    """
    incoming_message_ids = set()
    while True:
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
            s.bind(("", PORT))
            s.setblocking(0)
            result = select.select([s], [], [])
            msg = result[0][0].recv(BUFFER_SIZE)
            base64_encoded_data = base64.b64encode(msg)
            decoded_image_data = base64.decodebytes(base64_encoded_data)
        # now we got the message
        try:  # check whether message format is valid
            incoming_message_in_json = json.loads(decoded_image_data)
        except Exception as e:  # if not raise exception and listen next message
            print(e)
            print("***Incoming message was not in good format!***")
            continue

        # return discover response
        if incoming_message_in_json["TYPE"] == 1:

            # do not listen to myself
            if incoming_message_in_json["IP"] == MY_IP:
                continue
            # only take one udp message from each burst
            if incoming_message_in_json["ID"] not in incoming_message_ids:
                incoming_message_ids.add(incoming_message_in_json["ID"])
                # print(incoming_message_in_json)
            else:
                continue
            discover_response_thread = threading.Thread(
                target=send_discover_response, args=(incoming_message_in_json["IP"],))
            discover_response_thread.start()
        elif incoming_message_in_json["TYPE"] == 4:  # a part  of the file came
            if incoming_message_in_json["SEQ"] == -1:
                print(UI_COLORS["GREEN"]+"\n\n*** YOU HAVE A NEW FILE WITH NAME: {}***".format(
                    "x_"+incoming_message_in_json["NAME"]) + UI_COLORS["END"])
                send_acknowledgement(
                    incoming_message_in_json["IP"], incoming_message_in_json["NAME"], -1)
                print_onlines("only print")
            else:
                chunk_file = open("x_"+incoming_message_in_json["NAME"], 'ab+')
                chunk_file.write(base64.decodebytes(
                    incoming_message_in_json["DATA"].encode('utf-8')))
                chunk_file.close()
                send_acknowledgement(
                    incoming_message_in_json["IP"], incoming_message_in_json["NAME"], incoming_message_in_json["SEQ"])


if __name__ == "__main__":
    NAME = input("Enter your name:\t")

    tcp_server__for_chat_and_discover_response_thread = threading.Thread(
        target=initialize_tcp_server)
    udp_server_listen_thread = threading.Thread(
        target=initialize_udp_server_listen)
    discover_thread = threading.Thread(target=send_discover)

    tcp_server__for_chat_and_discover_response_thread.start()
    udp_server_listen_thread.start()
    discover_thread.start()
