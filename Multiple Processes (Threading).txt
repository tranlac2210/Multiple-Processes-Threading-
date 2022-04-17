/*
Author: Lac Tran
Course: COSC 3360
Description:
    Creating a parallel fixed-length code decompressor using the 
    tools we learned in class to create multiple processes and threads.

*/
#include <iostream>
#include <string>
#include <pthread.h>
#include <cmath>
#include <bits/stdc++.h>
#include <vector>
#include <algorithm>
#include <bits/stdc++.h> // for reverse function
#include <sstream>  
#include <unistd.h> //sleep
#include <stdlib.h>


using namespace std;

vector< pair <string, int> > character_freq_vector;
vector< pair <string, string> > character_binary_vector;

struct Data
{
    string symbol_char = "";
    int decimal_value = 0;
    string binary_value = "";
    string compressed_mess = "";
    int greatest_bit = 0;
};

struct Data2
{
    int tracking_index = 0;
    string compressed_mess = "";
    int greatest_bit = 0;
    vector< pair <string, string> > character_binary_vector;
};

string decompress(vector< pair <string, string> > character_binary_vector, string processing_message, int greatest_bit)
{
    string message = "";
    string char_binary = processing_message.substr(0, greatest_bit);

    for(int i=0; i < character_binary_vector.size(); i++)
    {
        if (character_binary_vector[i].second == char_binary){
            message = character_binary_vector[i].first;
            break;
        }
    }
    return message;
}

string decimal_to_binary(int decimal_num, int greatest_bit)
{
    string bit_result = "";
    int remaining = 0;
    while (decimal_num != 0)
    {
        remaining = decimal_num % 2;
        decimal_num /= 2;
        bit_result += to_string(remaining);
    }
    reverse(bit_result.begin(),bit_result.end());
    
    if (bit_result.length() < greatest_bit)
    {
        int num_missing = greatest_bit - bit_result.length();
        string temp = "";
        for (int i = 0; i < num_missing; i++){
            temp += "0";
        }
        temp += bit_result;
        bit_result = temp;
    }

    return bit_result;
}

int convert_string_int(string str){
    // Initialize a variable 
    int result = 0; 
    int str_length = str.length(); 
  
    // Iterate till length of the string 
    for (int i = 0; i < str_length; i++)
    { 
        result = result * 10 + (int(str[i]) - 48); 
    }
    return result;
}


void *n_child_thread(void *data)
{
    //Child thread process to store data in local variable itseft
    struct Data *vdata = (struct Data*)data;
    string symbol_char = vdata->symbol_char;
    int decimal_value = vdata->decimal_value;
    string compressed_mess = vdata->compressed_mess;
    int greatest_bit = vdata->greatest_bit;
    int freq = 0;

    string result = decimal_to_binary(decimal_value, greatest_bit);
    for (unsigned i = 0; i < compressed_mess.length(); i += greatest_bit) 
    {
        if (compressed_mess.substr(i, greatest_bit) == result)
        {
            freq++;
        }
    }
    
    cout << "Character: " << symbol_char << ", Code: " << result << ", Frequency: " << freq << endl;
    character_freq_vector.push_back(make_pair(symbol_char, freq));
    character_binary_vector.push_back(make_pair(symbol_char, result));
    return NULL;
}

void *m_child_thread(void *data)
{
    //Child thread process to store data in local variable itseft
    struct Data2 *vdata = (struct Data2*)data;
    int greatest_bit = vdata->greatest_bit;
    string compressed_mess = vdata->compressed_mess;
    vector< pair <string, string> > character_binary_vector = vdata->character_binary_vector;
    int tracking_index = vdata->tracking_index;
    string result = "";

    string char_binary = compressed_mess.substr(tracking_index * greatest_bit, greatest_bit);
    for(int i=0; i < character_binary_vector.size(); i++)
    {
        if (character_binary_vector[i].second == char_binary){
            result = character_binary_vector[i].first;
            break;
        }
    }
    cout << result;
    return NULL;
}

int main()
{   
    int NTHREADS = 0, 
        MTHREADS = 0, 
        symbol_code_temp = 0,
        largest_decimal = 0,
        greatest_bit = 0,
        line_counter = 1;
    string line = "",
        binary_code = "",
        character_temp = "",
        compressed_mess = "";
    vector< pair <string, int> > character_decimal_vector;

    while(getline(cin, line))
    {
        // Detect the first line
        if (line_counter == 1){
            NTHREADS = convert_string_int(line);
            line_counter++;
            continue;
        }
        // Detect the last line 
        if (line_counter == (NTHREADS + 2)){
            compressed_mess = line;
            break;
        }
        int space_position = 0;
        string temp = "";
        
        //Determine the space symbol between our symbol
        //and the binary code. Then, processing line by line
        //to extract the symbol and its binary code 
        //coresponding.
        if(line[0] != ' ')
        {
            space_position = line.find(" ");
            character_temp = line.substr(0,space_position);
        }
        //In a special case when symbol is a white space,
        //we may process the line a little different
        else
        {
            space_position = line.find(" ") + 1;
            character_temp = " ";
        }
        
        symbol_code_temp = stoi(line.substr(space_position+1));
        largest_decimal = max(largest_decimal, symbol_code_temp);
        character_decimal_vector.push_back(make_pair(character_temp, symbol_code_temp));
        line_counter++;
    }
    static struct Data *data = new Data[NTHREADS];

    greatest_bit = ceil(log2(largest_decimal + 1));
    cout << "Alphabet:" << endl;

    //Initialize pthread id
    pthread_t tid1[NTHREADS - 1];

    //Creating child thread
    for(int i = 0; i < NTHREADS; i++)
    {
        data[i].symbol_char = character_decimal_vector[i].first;
        data[i].decimal_value = character_decimal_vector[i].second;
        data[i].greatest_bit = greatest_bit;
        data[i].compressed_mess = compressed_mess;

        //Create n child threads
        if(pthread_create(&tid1[i], NULL, n_child_thread, (&data[i])))
        {
            fprintf(stderr, "Error creating thread\n");
			return 1;
        }
        sleep(0.5);
    }

    //Wait for the other threads to finish.
    for(int i = 0; i < NTHREADS; i++)
        pthread_join(tid1[i], NULL);

    delete []data;

    // Determine the total number of MTHREADS 
    for(int i=0; i < character_freq_vector.size(); i++)
    {
        MTHREADS += character_freq_vector[i].second;
    }

    cout << endl;
    cout << "Decompressed message: ";
    //Initialize pthread id
    pthread_t tid2[MTHREADS - 1];
    static struct Data2 *data2 = new Data2[MTHREADS];

    //Creating child thread
    for(int i = 0; i < MTHREADS; i++)
    {
        //Prepare data for each child thread which are symbol and binary code
        data2[i].greatest_bit = greatest_bit;
        data2[i].compressed_mess = compressed_mess;
        data2[i].character_binary_vector = character_binary_vector;
        data2[i].tracking_index = i;

        //Create n child threads
        if(pthread_create(&tid2[i], NULL, m_child_thread, (&data2[i])))
        {
            fprintf(stderr, "Error creating thread\n");
			return 1;
        }
        sleep(0.5);
    }

    //Wait for the other threads to finish.
    for(int i = 0; i < MTHREADS; i++)
        pthread_join(tid2[i], NULL);

    cout << endl;
    delete []data2;
    
    return 0;
}


