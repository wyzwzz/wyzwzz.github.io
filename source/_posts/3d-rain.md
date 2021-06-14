---
title: 3d_rain
date: 2021-06-14 21:08:22
tags:
---

```c++

#include<iostream>
#include<vector>
using namespace std; 
void calidx(vector<vector<int> >& heightMap,vector<vector<int> >& idxarr,int level)
    {
        for(int i=0;i<heightMap.size();i++)
        {
            for(int j=0;j<heightMap[0].size();j++)
            {
                if(i!=0 && j!=0 && i!=heightMap.size()-1 && j!=heightMap[0].size()-1)
                {
                    int tmp=0;
                    if(heightMap[i][j]>=heightMap[i-1][j])
                        tmp+=1;
                    if(heightMap[i][j]>=heightMap[i][j+1])
                        tmp+=2;
                    if(heightMap[i][j]>=heightMap[i+1][j])
                        tmp+=4;
                    if(heightMap[i][j]>=heightMap[i][j-1])
                        tmp+=8;
                    idxarr[i][j]=tmp;
                }
                if(heightMap[i][j]==level)
                	idxarr[i][j]=-1;
            }
        }
        bool reconduct=true;
    	bool intoloop=true;
        while(reconduct==true && intoloop==true)
	    {
	    	intoloop=false;
	    	reconduct=false;
	    	for(int i=1;i<heightMap.size()-1;i++)
	        {
	        	for(int j=1;j<heightMap[0].size()-1;j++)
	        	{
	        		intoloop=true;
	        		if(heightMap[i][j]>=heightMap[i-1][j] && idxarr[i-1][j]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
	        		else if(heightMap[i][j]>=heightMap[i+1][j] && idxarr[i+1][j]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
	        		else if(heightMap[i][j]>=heightMap[i][j-1] && idxarr[i][j-1]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
	        		else if(heightMap[i][j]>=heightMap[i][j+1] && idxarr[i][j+1]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
				}
			}
		}
    }
int trapRainWater(vector<vector<int> >& heightMap) {
    //up:1 right:2 down:4 left:8 dontflow:0
    vector<vector<int> > idxarr;
    idxarr.resize(heightMap.size());
    if(heightMap.size()==0) return 0;
    int maxlevel=0;
    for(int i=0;i<heightMap.size();i++)
    {

        for(int j=0;j<heightMap[0].size();j++)
        {
            if(heightMap[i][j]>maxlevel)
                maxlevel=heightMap[i][j];
            if(i==0 || j==0 || i==heightMap.size()-1 || j==heightMap[i].size()-1)
                idxarr[i].push_back(-1);
            else
            {
                int tmp=0;
                if(heightMap[i][j]>=heightMap[i-1][j])
                    tmp+=1;
                if(heightMap[i][j]>=heightMap[i][j+1])
                    tmp+=2;
                if(heightMap[i][j]>=heightMap[i+1][j])
                    tmp+=4;
                if(heightMap[i][j]>=heightMap[i][j-1])
                    tmp+=8;
                idxarr[i].push_back(tmp);
            }
        }
    }
     bool reconduct=true;
    	bool intoloop=true;
        while(reconduct==true && intoloop==true)
	    {
	    	intoloop=false;
	    	for(int i=1;i<heightMap.size()-1;i++)
	        {
	        	for(int j=1;j<heightMap[0].size()-1;j++)
	        	{
	        		intoloop=true;
	        		if(heightMap[i][j]==heightMap[i-1][j] && idxarr[i-1][j]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
	        		else if(heightMap[i][j]==heightMap[i+1][j] && idxarr[i+1][j]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
	        		else if(heightMap[i][j]==heightMap[i][j-1] && idxarr[i][j-1]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
	        		else if(heightMap[i][j]==heightMap[i][j+1] && idxarr[i][j+1]==-1 && idxarr[i][j]!=-1)
	        		{
	        			idxarr[i][j]=-1;
	        			reconduct=true;
					}
					else
					reconduct=false;
				}
			}
		}
    int rain=0, level=0;
    while(level<=maxlevel)
    {
    	cout<<"rian:"<<rain<<endl;

      
        for(int i=1;i<idxarr.size()-1;i++)
        {
            for(int j=1;j<idxarr[0].size()-1;j++)
            {
				if(idxarr[i][j]>0 &&heightMap[i][j]==level)
                {
                    if(idxarr[i][j]&1)
                    {
                        
                        if(idxarr[i-1][j]!=-1 )
                        {
                        	idxarr[i-1][j]&=11;//1011
                            idxarr[i][j]&=14;//1110
						}
                    }
                    if(idxarr[i][j]&2)
                    {
                        
                        if(idxarr[i][j+1]!=-1)
                        {
                        	idxarr[i][j+1]&=7;//0111
                            idxarr[i][j]&=13;//1101
						}
                    }
                    if(idxarr[i][j]&4)
                    {
                        
                        if(idxarr[i+1][j]!=-1)
                        {
                        	idxarr[i+1][j]&=14;//1110
                            idxarr[i][j]&=11;//1011
						}
                    }
                    if(idxarr[i][j]&8)
                    {
                        
                        if(idxarr[i][j-1]!=-1)
                        {
                        	idxarr[i][j-1]&14;//1110
                            idxarr[i][j]&=7;//0111
						}
                    }
                }
            }
        }
        for(int i=0;i<idxarr.size();i++)
		{
			for(int j=0;j<idxarr[0].size();j++)
            	{
            		cout<<idxarr[i][j]<<" ";
            		if(idxarr[i][j]==0)
            		{
            				heightMap[i][j]++;
            				rain++; 
					} 
				}
            cout<<endl;
		}
		
        for(int i=0;i<idxarr.size();i++)
		{
			for(int j=0;j<idxarr[0].size();j++)
            	cout<<heightMap[i][j]<<" ";
            cout<<endl;
    	}
    	
        calidx(heightMap,idxarr,level);
        level++;
        cout<<"after\n";
        for(int i=0;i<idxarr.size();i++)
		{
			for(int j=0;j<idxarr[0].size();j++)
            	cout<<idxarr[i][j]<<" ";
            cout<<endl;
    	}
    }

    return rain;
}
int main()
{
	vector<vector<int> > heightMap;
	heightMap.resize(5);
	for(int i=0;i<5;i++)
	{
		for(int j=0;j<4;j++)
		{
			int x;
			cin>>x;
			heightMap[i].push_back(x);
		}
	}
	int rain=trapRainWater(heightMap);
	cout<<rain;
	return 0;
}


```